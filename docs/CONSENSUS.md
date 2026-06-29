# esp-pbft — Consensus Algorithm (3-phase PBFT)

> **Status:** 🟡 Draft
> **Scope:** 3-phase state machine, quorum collection, log, timeouts, GC
> **Out of scope:** View-change (separate doc), crypto internals, transport

---

## 1. Overview

esp-pbft implements the standard **3-phase Practical Byzantine Fault Tolerance** protocol (Castro & Liskov, OSDI 1999) with `n = 7` nodes, `f = 2` Byzantine faults, and **quorum = 2f+1 = 5**.

The state machine executes once per request, on every replica, with the same logical steps:

```
   FREE → PRE_PREPARED → PREPARED → COMMITTED → EXECUTED → GC
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

This struct is the canonical `pbft_log_entry_t` (single source of truth: [MEMORY.md §2.3](./MEMORY.md)); CONSENSUS matches it exactly.

```c
typedef enum {
    PBFT_ENTRY_FREE = 0,
    PBFT_ENTRY_PRE_PREPARED,
    PBFT_ENTRY_PREPARED,
    PBFT_ENTRY_COMMITTED,
    PBFT_ENTRY_EXECUTED,
} pbft_entry_state_t;

typedef struct {
    uint64_t           sequence;            // assigned by primary
    uint32_t           view;                // view it was pre-prepared in
    uint8_t            digest[32];          // SHA-256 of request payload
    uint8_t            msg_type;            // request type tag
    uint16_t           payload_len;         // bytes used in payload[]
    uint8_t            payload[PBFT_TX_PAYLOAD_MAX];   // inline request (CHANGED — audit A5)
    pbft_entry_state_t state;               // FREE/PRE_PREPARED/PREPARED/COMMITTED/EXECUTED
    uint8_t            prepare_bitmask;     // bit i = replica i sent Prepare
    uint8_t            commit_bitmask;      // bit i = replica i sent Commit
    uint8_t            exec_bitmask;        // bit i = replica i confirmed Execute (checkpoint)
    bool               have_pre_prepare;    // this entry has a stored Pre-Prepare
    bool               own_prepare_sent;    // this node broadcast its Prepare
    bool               own_commit_sent;     // this node broadcast its Commit
    bool               is_checkpoint;       // sequence is a checkpoint boundary
    uint8_t            retry_count;         // retransmits so far (<= PBFT_MAX_RETRIES)
    uint64_t           state_entered_ms;    // timestamp of last state transition
    uint64_t           commit_received_ms;  // timestamp first Commit observed
} pbft_log_entry_t;   // ~360 B; static_assert(sizeof <= 384)
```

**Quorum counts** come from `__builtin_popcount(entry->prepare_bitmask)` etc.; there are NO
`prepare_count` / `commit_count` fields and NO `prepare_from[7]` / `commit_from[7]` arrays.
The commit callback is registered ONCE globally (see [API-REFERENCE.md §5](./API-REFERENCE.md)) —
there is no per-entry `app_cb`. The node-level state type `pbft_state_t` / `PBFT_STATE_*`
([API-REFERENCE.md §7.1](./API-REFERENCE.md)) is DISTINCT from `pbft_entry_state_t` /
`PBFT_ENTRY_*`; the empty entry state is `PBFT_ENTRY_FREE`.

**Static array:** `pbft_log[PBFT_LOG_MAX_ENTRIES]` = `100 × ~360 B` = **~36 KB** in BSS (see [MEMORY.md §2.3](./MEMORY.md)).

**Cross-view cleanup:** On view-change to view V, scan log and free any entries where `entry->view < V` and `entry->state < PBFT_ENTRY_COMMITTED` (uncommitted). The `prepare_bitmask`/`commit_bitmask` must be cleared and `own_*_sent` reset because the new view assigns fresh sequence numbers (see [VIEW-CHANGE.md §5.2](./VIEW-CHANGE.md)).

**Anti-replay across views:** The MAC input binds `view‖sequence‖type‖digest‖payload` (see [CRYPTO.md §5.1](./CRYPTO.md)). A Prepare broadcast in view V cannot be replayed in view V+1 because the view field in the MAC input differs. The receiving replica recomputes MAC with the new view — the old MAC will not verify.

**Sequence numbering (A7):** Sequence numbers are view-global and monotonic across view-changes. The first Pre-Prepare in view v+1 carries sequence `low_watermark + 1`.

### 3.2 Per-view state

```c
typedef struct {
    uint32_t      view;                 // current view number
    uint8_t       primary_id;           // = view % n
    uint64_t      next_sequence;        // primary's next-to-assign counter
    uint64_t      last_committed_seq;   // last seq that reached COMMITTED
    uint64_t      last_executed_seq;    // last seq whose app callback fired
    // NOTE: low_watermark and high_watermark live in pbft_watermark_state_t
    // (single-owner rule per MEMORY §2.4 / H3 fix). Access them via
    // pbft_watermark_get_low() / pbft_watermark_get_high().

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
                       │  PBFT_ENTRY_FREE                   │
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
    // 3. Check sequence number is in valid range:
    //    accept low_watermark < seq <= high_watermark
    //    (high_watermark = low_watermark + PBFT_LOG_MAX_ENTRIES)
    if (pp->sequence <= pbft_watermark_get_low() ||
        pp->sequence > pbft_watermark_get_high()) {
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
    entry->state = PBFT_ENTRY_PRE_PREPARED;
    entry->state_entered_ms = esp_timer_get_time() / 1000;

    // 6. Broadcast Prepare
    pbft_prepare_t prepare = {
        .view = pp->view,
        .sequence = pp->sequence,
        .digest = pp->digest,
    };
    pbft_send_prepare(&prepare);

    // 7. Record own Prepare
    entry->prepare_bitmask  = (uint8_t)(1u << my_node_id);
    entry->own_prepare_sent = true;
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

    // 3. Record this Prepare
    entry->prepare_bitmask |= (uint8_t)(1u << sender);

    // 4. Check if we have quorum
    if (__builtin_popcount(entry->prepare_bitmask) >= 2 * PBFT_F + 1 &&  // 5 for f=2
        entry->state == PBFT_ENTRY_PRE_PREPARED) {
        entry->state = PBFT_ENTRY_PREPARED;
        entry->state_entered_ms = esp_timer_get_time() / 1000;

        // Broadcast Commit
        pbft_commit_t commit = {
            .view = p->view,
            .sequence = p->sequence,
            .digest = p->digest,
        };
        pbft_send_commit(&commit);

        // Record own Commit
        entry->commit_bitmask  = (uint8_t)(1u << my_node_id);
        entry->own_commit_sent = true;
    }
}
```

### 4.4 Pseudocode: handle Commit

```c
void pbft_handle_commit(const pbft_commit_t* c, uint8_t sender) {
    pbft_log_entry_t* entry = pbft_log_find(c->view, c->sequence);
    if (!entry) return;

    // Record first Commit observation time
    if (entry->commit_bitmask == 0) {
        entry->commit_received_ms = esp_timer_get_time() / 1000;
    }

    // Skip if not in PREPARED state
    if (entry->state != PBFT_ENTRY_PREPARED) {
        // Still record commit (may need it for later recovery)
        entry->commit_bitmask |= (uint8_t)(1u << sender);
        return;
    }

    // Verify digest
    if (memcmp(entry->digest, c->digest, 32) != 0) {
        log_warn("Commit digest mismatch from %d", sender);
        return;
    }

    // Record commit
    entry->commit_bitmask |= (uint8_t)(1u << sender);

    // Quorum check
    if (__builtin_popcount(entry->commit_bitmask) >= 2 * PBFT_F + 1) {
        entry->state = PBFT_ENTRY_COMMITTED;
        entry->state_entered_ms = esp_timer_get_time() / 1000;
        view_state.last_committed_seq = MAX(
            view_state.last_committed_seq, entry->sequence);

        // Skip execution if this sequence was already executed (e.g. re-proposed
        // across a view-change) — prevents double execution (G16).
        if (entry->sequence > view_state.last_executed_seq) {
            // Fire the globally-registered commit callback (API-REFERENCE §5).
            pbft_commit_cb_t cb = pbft_get_commit_cb();   // single global callback
            if (cb) {
                pbft_tx_t tx = {
                    .payload = entry->payload,
                    .payload_len = entry->payload_len,
                    .tx_id = entry->digest,
                };
                cb(&tx, entry->sequence);
            }
            view_state.last_executed_seq = entry->sequence;
        }

        entry->state = PBFT_ENTRY_EXECUTED;
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

**Constraint:** `next_sequence > pbft_watermark_get_high()` always (otherwise we exhaust log).

### 5.2 Handle submit() (primary path)

`pbft_submit` takes a NON-const `tx` and writes back `tx->digest` and `tx->sequence` as OUT fields (G17).

```c
esp_err_t pbft_submit(pbft_tx_t* tx) {
    if (my_node_id != view_state.primary_id) {
        // Non-primary: forward to primary (or reject — TBD)
        log_warn("Non-primary cannot submit directly");
        return PBFT_ERR_NOT_PRIMARY;
    }

    // 1. Compute digest (plain SHA-256 of the buffer, see CRYPTO.md / G18)
    pbft_sha256(tx->payload, tx->payload_len, tx->digest);

    // 2. Assign sequence
    uint64_t seq = view_state.next_sequence++;
    tx->sequence = seq;   // write back OUT field

    // 3. Build and broadcast Pre-Prepare
    pbft_pre_prepare_t pp = {
        .view = view_state.view,
        .sequence = seq,
        .digest = tx->digest,
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
        if (entry->state == PBFT_ENTRY_FREE) continue;

        uint64_t elapsed = now_ms - entry->state_entered_ms;

        if (entry->state == PBFT_ENTRY_PRE_PREPARED &&
            elapsed > view_state.prepare_timeout_ms) {
            log_warn("Prepare timeout for seq %llu", entry->sequence);
            pbft_initiate_view_change();
            return;
        }
        // ... similar for PREPARED, COMMITTED-without-execute
    }
}
```

### 6.3 Retransmit policy (before view-change)

Timeouts (§6.1) trigger view-change, but a transient packet loss should not cause view-change. We **retry** the local message up to `PBFT_MAX_RETRIES = 3` times before declaring failure, tracked by the per-entry `retry_count` counter (G8).

```c
#define PBFT_MAX_RETRIES  3
```

| Phase | Local action on timeout | Retries before view-change |
|-------|------------------------|----------------------------|
| Pre-Prepare → Prepare | Re-broadcast Prepare | up to 3 retries every `timeout_ms / 2`; full view-change at `timeout_ms` |
| Prepare → Commit | Re-broadcast Commit | up to 3 retries every `timeout_ms / 2`; full view-change at `timeout_ms` |
| Commit → Execute | Re-fire commit callback (no-op if already executed) | up to 3 retries every `timeout_ms / 2` |

**Why not infinite retries?** Each round-trip is ~10 ms; `PBFT_MAX_RETRIES = 3` rebroadcasts cover transient RF interference. Beyond that, the primary is likely faulty and view-change is the correct response.

```c
// Combined timeout + retransmit
void pbft_check_timeouts(void) {
    uint64_t now_ms = esp_timer_get_time() / 1000;

    for (int i = 0; i < PBFT_LOG_MAX_ENTRIES; i++) {
        pbft_log_entry_t* entry = &pbft_log[i];
        if (entry->state == PBFT_ENTRY_FREE) continue;

        uint64_t elapsed = now_ms - entry->state_entered_ms;

        // Before full timeout: retry local broadcast up to PBFT_MAX_RETRIES,
        // once per timeout_ms/2 window while quorum is missing.
        if (elapsed > view_state.prepare_timeout_ms / 2 &&
            entry->retry_count < PBFT_MAX_RETRIES) {
            if (entry->state == PBFT_ENTRY_PRE_PREPARED) {
                pbft_resend_prepare(entry);  // re-send our Prepare
                entry->retry_count++;
                continue;
            }
            if (entry->state == PBFT_ENTRY_PREPARED) {
                pbft_resend_commit(entry);   // re-send our Commit
                entry->retry_count++;
                continue;
            }
        }

        // Full timeout → view-change
        if (entry->state == PBFT_ENTRY_PRE_PREPARED &&
            elapsed > view_state.prepare_timeout_ms) {
            log_warn("Prepare timeout (no quorum) for seq %llu", entry->sequence);
            pbft_initiate_view_change();
            return;
        }
        // ... similar for PREPARED
    }
}
```

`retry_count` is cleared on each state transition (so each phase gets its own retry budget of `PBFT_MAX_RETRIES`).

### 6.4 Adaptive timeout (future enhancement)

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

Each log entry has a single `uint8_t` **bitmask** (`prepare_bitmask` or `commit_bitmask`), one bit per node. Quorum check is `__builtin_popcount(bitmask) >= 5`.

**Layout:** a single `uint8_t` (1 byte) holds all 7 nodes' bits — no `[7]` arrays.

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
    pbft_watermark_set_low(stable_seq);
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
    return next_sequence > pbft_watermark_get_high();
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

### 9.1 Throughput model

Three independent bottleneck dimensions:

| Dimension | ESP-NOW (250 B cap) | Wi-Fi UDP (1500 B MTU) |
|-----------|---------------------|------------------------|
| **Round-trip count** | 3 (Pre-Prep / Prep / Commit) | 3 (Pre-Prep / Prep / Commit) |
| **Per-TX broadcast cost** | 1 Pre-Prep + 1 Prep + 1 Commit = 3 packets × 7 receivers = 21 Tx | same shape, but TCP-friendly payload up to 1500 B |
| **Latency floor (no contention)** | ~30 ms per round | ~30 ms per round |
| **Max TPS (sequential, f=0 loss)** | ⌊1000/30⌋ = **33 TPS** | ⌊1000/30⌋ = **33 TPS** |
| **Max TPS with 5% packet loss (1 retry)** | ⌊1000/45⌋ ≈ **22 TPS** | ⌊1000/45⌋ ≈ **22 TPS** |

**Why TPS is the same on both transports:** PBFT's 3-phase round-trips dominate; transport bandwidth is irrelevant below 256 B TX. UDP only helps for **larger payloads** (where 250 B cap forces fragmentation) or **batching** (one Pre-Prepare carrying multiple TXs).

### 9.2 Batching (Phase 2)

If app can submit TXs in batches, primary can pack N TXs into one Pre-Prepare (up to `PBFT_TX_PAYLOAD_MAX = 256 B`, so N ≈ 3 TXs at 80 B each). New-View messages can carry longer O-set lists.

With **per-view batching** of N=4 TXs (one Pre-Prepare carries 4 digests + payloads):

| Phase | Per-TX latency | Cluster TPS |
|-------|----------------|-------------|
| No batching | 30 ms | 33 TPS |
| Batched (N=4) | 30 ms / 4 = 7.5 ms | **133 TPS** |

This requires changing the wire format to a "batch Pre-Prepare" (out of scope for v1; planned v2 — see [ROADMAP.md §3 P2](./ROADMAP.md)).

### 9.3 Pipelining

Not implemented in v1. With pipelining, the primary can assign sequence N+1 immediately after sending Pre-Prepare for N, without waiting for Commit. This pushes throughput toward the network-bound limit. **Decision:** Skip in v1; the 33 TPS single-TX baseline is sufficient for v1 use cases (sensor fusion, LED control).

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
| `pbft_log_entry_t` | ~360 B | inline 256 B payload + state + bitmasks |
| `PBFT_LOG_MAX_ENTRIES` | 100 | static array |
| **Total PBFT log** | **~36 KB** | BSS, see [MEMORY.md §2.3](./MEMORY.md) |
| `pbft_view_state_t` | ~120 B | per-view, single instance |
| `pbft_dedup_entry_t` | 44 B | dedup cache, 64 entries |
| **Total dedup cache** | **~2.8 KB** | BSS, see §15 |

The inline-payload size is required by the no-heap design (see ARCHITECTURE.md §11). Earlier docs quoted "~120 B per entry" assuming pointer-to-payload; that figure is superseded.

---

## 14. Concurrency assumptions (v1)

- **Single-threaded** PBFT state machine (see [ARCHITECTURE.md §6](./ARCHITECTURE.md))
- Network receive callback enqueues to PBFT queue
- Timer callback runs in same task
- App `submit()` runs in app task — must use `pbft_queue_submit()` if calling from another task

**Thread-safety:** see [API-REFERENCE.md](./API-REFERENCE.md) (when written).

---

## 15. Duplicate-request defense (TX_FLAG_DEDUP)

### 15.1 Why needed

If the app calls `pbft_submit(tx)` twice (e.g., due to retry after timeout), the same TX payload will be assigned two different sequences by the primary. Without dedup, both execute — wasting log slots and producing two callbacks.

### 15.2 Mechanism

Primary maintains a **TX dedup cache**: `digest → last_sequence` map, 64 entries, FIFO eviction.

```c
#define PBFT_DEDUP_SIZE  64   // BSS, ~2 KB

typedef struct {
    uint8_t  digest[32];
    uint64_t seq;
    uint32_t timestamp_ms;   // for FIFO eviction
} pbft_dedup_entry_t;

static pbft_dedup_entry_t s_dedup[PBFT_DEDUP_SIZE];

esp_err_t pbft_dedup_check_and_insert(const uint8_t digest[32],
                                       uint64_t new_seq,
                                       uint32_t now_ms) {
    // Lookup
    for (int i = 0; i < PBFT_DEDUP_SIZE; i++) {
        if (memcmp(s_dedup[i].digest, digest, 32) == 0) {
            // Duplicate — return existing seq (caller can decide to drop or re-route)
            return PBFT_ERR_DUPLICATE;  // carries s_dedup[i].seq via out-param
        }
    }
    // Insert at oldest slot
    int oldest = 0;
    for (int i = 1; i < PBFT_DEDUP_SIZE; i++) {
        if (s_dedup[i].timestamp_ms < s_dedup[oldest].timestamp_ms) oldest = i;
    }
    memcpy(s_dedup[oldest].digest, digest, 32);
    s_dedup[oldest].seq = new_seq;
    s_dedup[oldest].timestamp_ms = now_ms;
    return ESP_OK;
}
```

**Memory cost:** 64 × (32 + 8 + 4) = 64 × 44 B = **~2.8 KB** in BSS.

### 15.3 Caller contract (see API-REFERENCE §4)

App can opt in to dedup:

```c
pbft_submit_ex(tx, PBFT_TX_FLAG_DEDUP);   // dedup-aware
pbft_submit(tx);                           // may produce duplicates
```

When dedup returns `PBFT_ERR_DUPLICATE`, app should **not retry** — the original TX is in-flight or committed. Returning the existing seq is informational (debug aid).

### 15.4 Replicas do NOT dedup

Only the **primary** runs the dedup check. Replicas accept any Pre-Prepare from the primary — that's PBFT's contract. If primary is Byzantine and replays a committed TX, replicas see `(view, seq)` as the ordering key, not the digest, so they're free to re-execute. The application's app callback should be **idempotent** for this reason (see [DEPLOYMENT.md §4](./DEPLOYMENT.md)).

---

## 16. Open questions (consensus-level)

| # | Question | Status |
|---|----------|--------|
| C1 | Adaptive timeout vs fixed 1000 ms? | ✅ Fixed for v1; adaptive planned v2 (§6.4) |
| C2 | How to handle duplicate submissions? (same digest, new seq) | ✅ Dedup cache, §15 |
| C3 | Should primary broadcast Commit immediately after Prepare? | 🟡 TBD (optimisation) |
| C4 | What if app_callback() blocks for long? | ✅ v1: must be non-blocking (API-REFERENCE §5) |

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