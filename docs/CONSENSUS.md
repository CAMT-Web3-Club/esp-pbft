# esp-pbft — Consensus Algorithm (3-phase PBFT)

> **Status:** 🟡 Draft
> **Scope:** 3-phase state machine, quorum collection, log, timeouts, GC
> **Out of scope:** View-change (separate doc), crypto internals, transport

---

## 1. Overview

esp-pbft implements the standard **3-phase Practical Byzantine Fault Tolerance** protocol (Castro & Liskov, OSDI 1999) with `n = 7` nodes, `f = 2` Byzantine faults, and **quorum = 2f+1 = 5**.

The state machine executes once per request, on every replica, with the same logical steps:

```
   IDLE → PRE_PREPARED → PREPARED → COMMITTED → EXECUTED → GC
```

Primary is determined by `p = view mod n`. View-change rotates the primary when faulty behavior is detected.

---

## 2. Roles and primary selection

### 2.1 Per-node role

Each node has exactly one role per view:

| Role | Count | Behavior |
|------|-------|----------|
| **Primary** | 1 (`view mod n`) | Assigns sequence numbers, broadcasts Pre-Prepare |
| **Replica** | n−1 = 6 | Validates Pre-Prepare, broadcasts Prepare and Commit |

### 2.2 Primary formula

```c
static inline uint8_t pbft_primary_of(uint32_t view) {
    return (uint8_t)(view % PBFT_CLUSTER_SIZE);
}
```

### 2.3 Rotating primary

```
view = 0 → primary = 0
view = 1 → primary = 1
view = 2 → primary = 2
...
view = 6 → primary = 6
view = 7 → primary = 0   (cycle)
```

---

## 3. Data structures

### 3.1 Per-request entry (in PBFT log)

```c
typedef struct {
    uint32_t   view;                  // assigned by primary
    uint64_t   sequence;              // assigned by primary (monotonic)
    uint8_t    digest[32];            // SHA-256 of TX payload
    uint8_t    state;                 // IDLE / PRE_PREPARED / PREPARED / COMMITTED / EXECUTED
    uint8_t    primary_id;            // who assigned this sequence

    // Pre-Prepare cache
    uint8_t    have_pre_prepare;      // boolean
    uint8_t*   payload;               // TX data, 0..PBFT_TX_PAYLOAD_MAX
    size_t     payload_len;

    // Prepare certificate (≥2f+1 matching Prepare messages)
    uint8_t    prepare_count;         // 0..7
    uint8_t    prepare_from[7];       // bitmask of replicas who sent Prepare

    // Commit certificate (≥2f+1 matching Commit messages)
    uint8_t    commit_count;
    uint8_t    commit_from[7];        // bitmask

    // App callback (fired at COMMITTED)
    pbft_commit_cb_t app_cb;          // NULL = no callback registered

    // Sequence number garbage collection
    uint8_t    gc_marked;             // true after stable checkpoint
} pbft_log_entry_t;
```

**Static array:** `pbft_log[PBFT_LOG_MAX_ENTRIES]` = `100 × ~120 B` = **~12 KB** in BSS (see [ARCHITECTURE.md §11](./ARCHITECTURE.md)).

### 3.2 Per-view state

```c
typedef struct {
    uint32_t      view;                 // current view number
    uint8_t       primary_id;           // = view % n
    uint64_t      next_sequence;        // primary's next-to-assign counter
    uint64_t      last_committed_seq;   // last seq that reached COMMITTED
    uint64_t      last_executed_seq;    // last seq whose app callback fired
    uint64_t      low_watermark;        // GC: entries below this can be reclaimed
    uint64_t      high_watermark;       // GC: pre-prepare allowed above this

    // Timeouts
    uint32_t      prepare_timeout_ms;   // default 1000
    uint32_t      commit_timeout_ms;    // default 1000
    uint32_t      viewchange_timeout_ms;// default 2000
} pbft_view_state_t;
```

---

## 4. State machine (per request, per replica)

### 4.1 ASCII diagram

```
                            receive Pre-Prepare(v, n, d, payload)
                                       │
                                       ▼
                       ┌─────────────────────────────────────┐
                       │  IDLE                              │
                       │  (entry created in PBFT log)       │
                       └─────────────────┬───────────────────┘
                                         │
                                         │ verify HMAC + accept Pre-Prepare
                                         │ → broadcast Prepare(v, n, d)
                                         │
                                         ▼
                       ┌─────────────────────────────────────┐
                       │  PRE_PREPARED                      │
                       │  (own Prepare sent)                │
                       └─────────────────┬───────────────────┘
                                         │
                                         │ collect ≥2f+1 = 5 matching Prepare
                                         │ (incl. own)
                                         │
                                         ▼
                       ┌─────────────────────────────────────┐
                       │  PREPARED                          │
                       │  → broadcast Commit(v, n, d)       │
                       └─────────────────┬───────────────────┘
                                         │
                                         │ collect ≥2f+1 = 5 matching Commit
                                         │ (incl. own)
                                         │
                                         ▼
                       ┌─────────────────────────────────────┐
                       │  COMMITTED                         │
                       │  → call app callback (if registered)│
                       └─────────────────┬───────────────────┘
                                         │
                                         │ app returned (or no callback)
                                         │
                                         ▼
                       ┌─────────────────────────────────────┐
                       │  EXECUTED                          │
                       │  → mark for GC at low_watermark    │
                       └─────────────────────────────────────┘

   Any state on timeout → see §6 (view-change)
   View-change during any state → see [VIEW-CHANGE.md](./VIEW-CHANGE.md)
```

### 4.2 Pseudocode: handle Pre-Prepare

```c
void pbft_handle_pre_prepare(const pbft_pre_prepare_t* pp, uint8_t sender) {
    // 1. Validate sender is primary for this view
    if (sender != pp->primary_id) {
        log_warn("Pre-Prepare from non-primary %d", sender);
        return;
    }

    // 2. Verify HMAC (already done by transport layer)
    // 3. Check sequence number is in valid range
    if (pp->sequence < view_state.high_watermark ||
        pp->sequence > view_state.high_watermark + PBFT_LOG_MAX_ENTRIES) {
        log_warn("Pre-Prepare seq %llu out of range", pp->sequence);
        return;
    }

    // 4. Look up or create log entry
    pbft_log_entry_t* entry = pbft_log_get_or_create(pp->view, pp->sequence);
    if (!entry) return;  // log full

    // 5. Store payload and digest
    entry->have_pre_prepare = 1;
    entry->view = pp->view;
    entry->sequence = pp->sequence;
    memcpy(entry->digest, pp->digest, 32);
    entry->payload_len = pp->payload_len;
    memcpy(entry->payload, pp->payload, pp->payload_len);
    entry->state = PBFT_STATE_PRE_PREPARED;

    // 6. Broadcast Prepare
    pbft_prepare_t prepare = {
        .view = pp->view,
        .sequence = pp->sequence,
        .digest = pp->digest,
    };
    pbft_send_prepare(&prepare);

    // 7. Count own Prepare
    entry->prepare_from[my_node_id] = 1;
    entry->prepare_count = 1;
}
```

### 4.3 Pseudocode: handle Prepare

```c
void pbft_handle_prepare(const pbft_prepare_t* p, uint8_t sender) {
    // 1. Find log entry
    pbft_log_entry_t* entry = pbft_log_find(p->view, p->sequence);
    if (!entry) return;  // haven't seen Pre-Prepare yet

    // 2. Verify digest matches our Pre-Prepare
    if (memcmp(entry->digest, p->digest, 32) != 0) {
        log_warn("Prepare digest mismatch from %d", sender);
        return;
    }

    // 3. Count this Prepare
    if (!entry->prepare_from[sender]) {
        entry->prepare_from[sender] = 1;
        entry->prepare_count++;
    }

    // 4. Check if we have quorum
    if (entry->prepare_count >= 2 * PBFT_F + 1 &&  // 5 for f=2
        entry->state == PBFT_STATE_PRE_PREPARED) {
        entry->state = PBFT_STATE_PREPARED;

        // Broadcast Commit
        pbft_commit_t commit = {
            .view = p->view,
            .sequence = p->sequence,
            .digest = p->digest,
        };
        pbft_send_commit(&commit);

        // Count own Commit
        entry->commit_from[my_node_id] = 1;
        entry->commit_count = 1;
    }
}
```

### 4.4 Pseudocode: handle Commit

```c
void pbft_handle_commit(const pbft_commit_t* c, uint8_t sender) {
    pbft_log_entry_t* entry = pbft_log_find(c->view, c->sequence);
    if (!entry) return;

    // Skip if not in PREPARED state
    if (entry->state != PBFT_STATE_PREPARED) {
        // Still count commit (may need it for later recovery)
        if (!entry->commit_from[sender]) {
            entry->commit_from[sender] = 1;
            entry->commit_count++;
        }
        return;
    }

    // Verify digest
    if (memcmp(entry->digest, c->digest, 32) != 0) {
        log_warn("Commit digest mismatch from %d", sender);
        return;
    }

    // Count commit
    if (!entry->commit_from[sender]) {
        entry->commit_from[sender] = 1;
        entry->commit_count++;
    }

    // Quorum check
    if (entry->commit_count >= 2 * PBFT_F + 1) {
        entry->state = PBFT_STATE_COMMITTED;
        view_state.last_committed_seq = MAX(
            view_state.last_committed_seq, entry->sequence);

        // Fire app callback (in single-threaded mode, here is fine)
        if (entry->app_cb) {
            pbft_tx_t tx = {
                .payload = entry->payload,
                .payload_len = entry->payload_len,
                .tx_id = entry->digest,
            };
            entry->app_cb(&tx, entry->sequence);
        }

        entry->state = PBFT_STATE_EXECUTED;
    }
}
```

---

## 5. Primary's view (asymmetric role)

### 5.1 Sequence assignment

```c
uint64_t pbft_primary_assign_sequence(void) {
    return view_state.next_sequence++;
}
```

**Constraint:** `next_sequence > high_watermark` always (otherwise we exhaust log).

### 5.2 Handle submit() (primary path)

```c
esp_err_t pbft_submit(const pbft_tx_t* tx) {
    if (my_node_id != view_state.primary_id) {
        // Non-primary: forward to primary (or reject — TBD)
        log_warn("Non-primary cannot submit directly");
        return PBFT_ERR_NOT_PRIMARY;
    }

    // 1. Compute digest
    uint8_t digest[32];
    pbft_sha256(tx->payload, tx->payload_len, digest);

    // 2. Assign sequence
    uint64_t seq = view_state.next_sequence++;

    // 3. Build and broadcast Pre-Prepare
    pbft_pre_prepare_t pp = {
        .view = view_state.view,
        .sequence = seq,
        .digest = digest,
        .payload_len = tx->payload_len,
        .payload = tx->payload,  // borrowed pointer
        .primary_id = my_node_id,
    };
    pbft_send_pre_prepare(&pp);

    return ESP_OK;
}
```

### 5.3 Pre-Prepare assignment guarantee

The primary **must** assign sequence numbers in **monotonically increasing order**. If the primary skips a sequence (e.g., assigns 10 then 12, skipping 11), replicas reject — this is the "gap-free assignment" requirement of PBFT.

---

## 6. Timeout handling

### 6.1 Timeouts per phase

| Phase | Timeout | Default | Action on expiry |
|-------|---------|---------|------------------|
| Pre-Prepare wait | `prepare_timeout_ms` | 1000 ms | Send View-Change |
| Prepare wait | `prepare_timeout_ms` | 1000 ms | Send View-Change |
| Commit wait | `commit_timeout_ms` | 1000 ms | Send View-Change |
| New-View wait | `viewchange_timeout_ms` | 2000 ms | Send View-Change (again) |

### 6.2 Timeout detection (per request)

```c
// For each entry, track when it entered PRE_PREPARED state
// On timer tick, check all entries in non-terminal state
void pbft_check_timeouts(void) {
    uint64_t now_ms = esp_timer_get_time() / 1000;

    for (int i = 0; i < PBFT_LOG_MAX_ENTRIES; i++) {
        pbft_log_entry_t* entry = &pbft_log[i];
        if (entry->state == PBFT_STATE_FREE) continue;

        uint64_t elapsed = now_ms - entry->state_entered_ms;

        if (entry->state == PBFT_STATE_PRE_PREPARED &&
            elapsed > view_state.prepare_timeout_ms) {
            log_warn("Prepare timeout for seq %llu", entry->sequence);
            pbft_initiate_view_change();
            return;
        }
        // ... similar for PREPARED, COMMITTED-without-execute
    }
}
```

### 6.3 Adaptive timeout (future enhancement)

Initial value: 1000 ms. **Adaptive per round:**
- After every Prepare timeout: timeout *= 1.25 (capped at 5000 ms)
- After successful commit: timeout = max(1000, timeout * 0.9)

This avoids flapping during transient network slowness but recovers quickly when primary is truly faulty.

---

## 7. Quorum certificate collection

### 7.1 Definitions

```c
#define PBFT_F              2
#define PBFT_QUORUM         (2 * PBFT_F + 1)   // = 5
#define PBFT_CLUSTER_SIZE   7
```

### 7.2 Counting strategy

Each log entry has a **bitmask** (`prepare_from[7]` or `commit_from[7]`), one bit per node. Quorum check is `count_ones(bitmask) >= 5`.

**Optimization:** Use `uint8_t` (1 byte fits 8 nodes) or `uint32_t` (32 bits for future expansion).

```c
static inline uint8_t pbft_count_set_bits(uint8_t bm) {
    return __builtin_popcount(bm);  // GCC builtin, O(1)
}

bool pbft_has_quorum(uint8_t bitmask) {
    return pbft_count_set_bits(bitmask) >= PBFT_QUORUM;
}
```

### 7.3 Byzantine fault tolerance math

| Quorum (5) | Byzantine (≤2) | Property |
|------------|----------------|----------|
| At least 5 honest vote | At most 2 are faulty | ✅ at least 3 honest agree |
| 2 quorums overlap in | n - quorum = 7 - 5 = 2 nodes | Need ≥1 honest |
| Wait — overlap should be ≥ f+1 = 3 | overlap is exactly 2 + 1 = 3 | ✅ PBFT safety holds |

---

## 8. Log management

### 8.1 Log lifecycle

```
   free → allocated (Pre-Prepare arrives) → state machine → GC
                                                                  │
                                                                  ▼
                                                       recycled when low_watermark passes
```

### 8.2 GC triggers (low/high watermark)

| Watermark | Meaning |
|-----------|---------|
| `low_watermark` | Entries below this can be reclaimed (state is EXECUTED + stable checkpoint covers them) |
| `high_watermark` | Entries above this can NOT be pre-prepared (would exhaust log) |

**Constraint:** `low_watermark ≤ last_executed_seq ≤ next_sequence ≤ high_watermark + PBFT_LOG_MAX_ENTRIES`

### 8.3 GC implementation (on stable checkpoint)

```c
// Called when CHECKPOINT(seq, digest) reaches quorum
void pbft_gc_up_to(uint64_t stable_seq) {
    view_state.low_watermark = stable_seq;
    // Mark log entries below stable_seq as gc_marked
    for (int i = 0; i < PBFT_LOG_MAX_ENTRIES; i++) {
        if (pbft_log[i].sequence <= stable_seq &&
            pbft_log[i].state == PBFT_STATE_EXECUTED) {
            pbft_log_free(&pbft_log[i]);  // reuse the slot
        }
    }
}
```

### 8.4 Log full detection

```c
bool pbft_log_full(void) {
    return next_sequence > high_watermark + PBFT_LOG_MAX_ENTRIES;
}
```

If log is full, primary must **stop assigning new sequences** until checkpoint advances `high_watermark`.

---

## 9. Request lifecycle (normal flow, end-to-end)

```
   t=0  App calls pbft_submit(tx) [primary only]
        │
        ▼
   t=ε  Primary: digest = SHA256(tx), seq = next_seq++
        │     Build Pre-Prepare(v, seq, digest, tx)
        │     broadcast (MAC sign, 5 µs)
        ▼
   t+~10ms  Replicas receive Pre-Prepare
            │  verify HMAC (5 µs)
            │  store in log, mark PRE_PREPARED
            │  broadcast Prepare(v, seq, digest)
            ▼
   t+~20ms  All nodes collect ≥5 matching Prepares
            │  (incl. own + 4 others)
            │  mark PREPARED, broadcast Commit
            ▼
   t+~30ms  All nodes collect ≥5 matching Commits
            │  mark COMMITTED
            │  fire app callback (if registered)
            │  mark EXECUTED
            ▼
   t+~30ms  Done — TX committed at all honest replicas
```

**End-to-end latency:** ~30-50 ms (in best case, no packet loss).

**Throughput:** ~30 / sec = ~30 TPS per cluster (limited by 3-phase round-trips).

---

## 10. Byzantine primary scenarios

### 10.1 Scenario: Primary assigns conflicting sequences

**Symptom:** Two different replicas receive Pre-Prepare with same `(view, seq)` but different `digest`.

**Defense:** Each replica compares digest with its own Pre-Prepare for that seq. Mismatch → log alarm + **don't prepare**. If 2f+1 honest replicas refuse to prepare, the sequence never reaches `prepared` state, and the request is dropped.

**Recovery:** After timeout, view-change → new primary re-proposes or skips.

### 10.2 Scenario: Primary stops proposing

**Symptom:** App calls `submit(tx)` but Pre-Prepare never sent.

**Defense:** None at submit time. After `next_sequence` stalls for >1000 ms, no Prepare arrives → replicas timeout → view-change.

### 10.3 Scenario: Primary sends Pre-Prepare with bad HMAC

**Symptom:** MAC verify fails on all replicas.

**Defense:** Replicas drop the message silently. View-change triggered by timeout.

### 10.4 Scenario: Replica equivocation (sends different Prepare to different peers)

**Symptom:** Two replicas receive same `(view, seq)` from sender X with different digests in Prepare.

**Defense:** Each replica counts **matching** Prepares (same digest). Equivocating replica can't contribute to quorum — at most 1 vote instead of 1 reliable vote. Quorum still achievable because 6 honest replicas contribute.

---

## 11. Request flow with malicious client

The client submits TX to **current primary only**. If client submits to non-primary:
- v1: `pbft_submit()` returns `PBFT_ERR_NOT_PRIMARY`
- App must retry with correct primary (or use `pbft_forward_to_primary()` helper)

**Why:** Allowing any node to accept client requests would require additional routing logic (out of scope for v1).

---

## 12. Failure detection

| Symptom | Detection | Action |
|---------|-----------|--------|
| No Pre-Prepare after `submit()` | `next_sequence` doesn't advance | view-change after timeout |
| Pre-Prepare received but no Prepare from peers | timer per entry | view-change after `prepare_timeout_ms` |
| Prepare received but no Commit | same timer | view-change after `commit_timeout_ms` |
| Commit received but no commit-certificate | timer | log warning + view-change |
| View-change sent but no New-View | timer | view-change (with higher view) |

---

## 13. Memory cost (per request state)

| Item | Size | Notes |
|------|------|-------|
| `pbft_log_entry_t` | ~120 B | payload (256 B max) + state + bitmasks |
| `PBFT_LOG_MAX_ENTRIES` | 100 | static array |
| **Total PBFT log** | **~12 KB** | BSS, see ARCHITECTURE.md §11 |
| `pbft_view_state_t` | ~80 B | per-view, single instance |

---

## 14. Concurrency assumptions (v1)

- **Single-threaded** PBFT state machine (see [ARCHITECTURE.md §6](./ARCHITECTURE.md))
- Network receive callback enqueues to PBFT queue
- Timer callback runs in same task
- App `submit()` runs in app task — must use `pbft_queue_submit()` if calling from another task

**Thread-safety:** see [API-REFERENCE.md](./API-REFERENCE.md) (when written).

---

## 15. Open questions (consensus-level)

| # | Question | Status |
|---|----------|--------|
| C1 | Adaptive timeout vs fixed 1000 ms? | 🟡 Adaptive recommended, fixed for v1 |
| C2 | How to handle duplicate submissions? (same digest, new seq) | 🟡 Reject duplicates within view |
| C3 | Should primary broadcast Commit immediately after Prepare? | 🟡 TBD (optimisation) |
| C4 | What if app_callback() blocks for long? | 🟡 v1: must be non-blocking |

---

## 16. References

- [HANDOVER.md](./HANDOVER.md) — n=7, f=2, quorum=5
- [ARCHITECTURE.md](./ARCHITECTURE.md) — state machine diagram, data flow
- [CRYPTO.md](./CRYPTO.md) — HMAC compute/verify
- [PROTOCOL.md](./PROTOCOL.md) — wire format
- [VIEW-CHANGE.md](./VIEW-CHANGE.md) — primary rotation (TBD)
- [CHECKPOINT.md](./CHECKPOINT.md) — GC triggering (TBD)
- Castro & Liskov, "Practical Byzantine Fault Tolerance" (OSDI 1999) — original paper

---

**End of CONSENSUS.md (v0.1 draft — awaiting review)**