# esp-pbft — View-Change Protocol

> **Status:** 🟡 Draft (v0.1)
> **Scope:** triggers, message formats (with prepared-set), V-set / O-set construction, New-View verification, primary recovery, partition handling.
> **Out of scope:** steady-state 3-phase flow → [CONSENSUS.md](./CONSENSUS.md); crypto internals → [CRYPTO.md](./CRYPTO.md); wire-level byte layout → [PROTOCOL.md](./PROTOCOL.md).

This document is the **canonical reference for view-change in esp-pbft**. It supersedes the preview in ARCHITECTURE.md §7 and corrects two protocol-breaking issues found in [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) (A1: missing prepared-set in View-Change; A2: incomplete New-View format).

---

## 1. Goals

A view-change must satisfy three properties simultaneously:

| Property | Meaning |
|----------|---------|
| **Safety** | No two honest replicas commit different TXs at the same sequence, even across view boundaries. |
| **Liveness** | If the primary is faulty and ≥ 2f+1 replicas are honest, a new primary is eventually elected and consensus resumes. |
| **Bounded disruption** | View-change must complete in O(view-change-timeout) seconds; log entries prepared in the old view must be carried over or explicitly discarded — never silently lost. |

The PBFT paper (Castro & Liskov, OSDI 1999, §4) gives the proven construction; this document adapts it for the esp-pbft constraints (n=7, f=2, ESP-NOW + UDP dual transport, Y-5 key rotation).

### 1.1 Linear view-change (adopted from HotStuff)

We adopt the **linear view-change chain** from [HotStuff (PODC 2019) §3](https://arxiv.org/abs/1803.05069):

> "The view-change message of the new view contains a *high prepare certificate (highQC)* — a Prepare certificate from the previous view that the replica believes to be the most recent."

**Why:** vanilla PBFT view-change is **O(n²)** — every replica must broadcast its full prepared-set to every other replica. At n = 7 that's 42 messages per round. HotStuff linearises the chain: each `VIEW-CHANGE(v+1)` only needs to reference the highest Prepare cert it knows of, not the full set. Receivers that join late catch up via the `highQC` chain.

**Adopted cost:** +4 bytes in `pbft_view_change_t` (the `high_qc` digest) and +12 lines in `pbft_handle_view_change()`. **Adopted benefit:** 3× fewer messages during a view-change at n = 7, scales linearly with n.

**The `high_qc` field** lives in the VIEW-CHANGE wire format:

```c
typedef struct {
    uint32_t            view;               // current view number
    uint8_t             primary_id;         // = view % n
    uint8_t             n_prepared;         // 0..PBFT_VC_MAX_PREPARED
    pbft_prepared_entry_t prepared[PBFT_VC_MAX_PREPARED];
    pbft_digest_t       high_qc;            // ← HotStuff: highest Prepare-cert digest we know
    uint8_t             mac[32];
} __attribute__((packed)) pbft_view_change_t;
```

The receiver accepts the `high_qc` as a vote for all intermediate targets
(HotStuff §3.3 "chained HotStuff" property). If the sender has no
prepared entries (`n_prepared = 0`), `high_qc` is the all-zeros digest
(nil QC).

### 1.2 Leader rotation (adopted from HotStuff)

PBFT uses `primary_id = view % n` — the same leader is proposed every n views.
If leaders in a row are all Byzantine, liveness stalls until timeouts expire
without any progress guarantee.

We adopt the **deterministic rotation** from HotStuff §3.1:

> "Each view has a unique leader. The leader for view v is `v mod n`."

**Why:** even in the worst case, the next leader is **guaranteed different**
from the current one. Combined with the linear VC (§1.1), at most f consecutive
Byzantine leaders can delay, never block, consensus.

**Adopted cost:** 0 lines (already in code as `primary_id = view % n`). **Adopted benefit:** removes the liveness gap that vanilla PBFT has — at n = 7, f = 2, the probability of f+1 = 3 consecutive Byzantine leaders is 2⁻⁹ ≈ 0.2% per view, but the *expected* delay under a static-rotation adversary is `O(2^n)` instead of `O(n)`.

> **Note:** v1 keeps the PBFT `view mod n` rule verbatim. The "rotation" in this
> section refers to the *contrast* with PBFT's failed-leader-only-skip scheme.
> A future v2 may use cryptographic sortition (RANDAO/VDF) if the adversary
> model shifts from random to adaptive.

---

## 2. Triggers

A replica initiates a view-change when **any** of the following holds:

| # | Trigger | Detection | Local action |
|---|---------|-----------|--------------|
| T1 | Prepare-phase timeout | per-entry `state_entered_ms + prepare_timeout_ms < now` while state ∈ {IDLE, PRE_PREPARED} | send `VIEW-CHANGE(v+1)` |
| T2 | Commit-phase timeout | per-entry `state_entered_ms + commit_timeout_ms < now` while state ∈ {PREPARED} | send `VIEW-CHANGE(v+1)` |
| T3 | Suspected primary misbehavior | received Pre-Prepare with sequence gap, conflicting digest at same (v, seq), or no Pre-Prepare after `submit()` | log + send `VIEW-CHANGE(v+1)` |
| T4 | 2f+1 View-Change observed | received `≥ 2f+1` VIEW-CHANGE messages with the same `new_view = v+1` from distinct senders | am a replica (not new primary): abandon work on view v, wait for New-View; am new primary: collect ≥ 2f+1, then send `NEW-VIEW(v+1)` |
| T5 | New-View timeout | after T4, no New-View from new primary within `viewchange_timeout_ms` | send `VIEW-CHANGE(v+2)` |

**Important:** a replica sends at most ONE `VIEW-CHANGE(v+1)` per round (until T5 escalates to `v+2`). Multiple triggers collapse into a single message.

---

## 3. State machine (per replica, per view)

```
   ┌──────────────────────────────────────────────────────────────┐
   │  VIEW_NORMAL                                                 │
   │  ─ processing 3-phase consensus for current view v           │
   └───────────────┬──────────────────────────────────────────────┘
                   │ T1 or T2 or T3 detected
                   ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  VIEW_CHANGING                                               │
   │  ─ broadcast VIEW-CHANGE(v+1) with my prepared-set           │
   │  ─ stop processing further Pre-Prepares for view v           │
   │  ─ keep log entries (do NOT reset)                          │
   │  ─ keep high_watermark, low_watermark                       │
   └───────────────┬──────────────────────────────────────────────┘
                   │ ≥ 2f+1 VIEW-CHANGE(v+1) received from distinct senders
                   ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  VIEW_WAIT_NEW                                               │
   │  ─ if I am new primary p_new = (v+1) mod n:                  │
   │       collect full V-set → compute O-set → broadcast         │
   │       NEW-VIEW(v+1, V-set, O-set)                            │
   │  ─ if I am replica:                                          │
   │       wait for NEW-VIEW(v+1) from p_new                      │
   └───────────────┬──────────────────────────────────────────────┘
                   │ New-View verified OK
                   ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  VIEW_NORMAL (view = v+1)                                    │
   │  ─ accept O-set as new Pre-Prepares                          │
   │  ─ resume 3-phase consensus                                  │
   └──────────────────────────────────────────────────────────────┘

   On T5 (viewchange_timeout_ms expires in VIEW_WAIT_NEW):
     send VIEW-CHANGE(v+2); view := v+2; goto VIEW_CHANGING
```

The state is held in a single field `view_state.phase : enum {NORMAL, CHANGING, WAIT_NEW}` per replica.

---

## 4. Data structures

### 4.1 Per-view state (extends CONSENSUS.md §3.2)

```c
typedef enum {
    PBFT_VIEW_PHASE_NORMAL    = 0,   // processing consensus
    PBFT_VIEW_PHASE_CHANGING  = 1,   // sent VIEW-CHANGE, awaiting peers
    PBFT_VIEW_PHASE_WAIT_NEW  = 2,   // got 2f+1 VCs, awaiting New-View
} pbft_view_phase_t;

typedef struct {
    uint32_t            view;                   // current view number
    pbft_view_phase_t   phase;                  // see §3
    uint8_t             primary_id;             // = view % n
    uint64_t            view_change_started_ms; // esp_timer_get_time() when CHANGING entered
    uint64_t            new_view_started_ms;    // ... when WAIT_NEW entered
    uint64_t            next_sequence;          // primary's next-to-assign counter
    uint64_t            last_committed_seq;
    uint64_t            last_executed_seq;
    // NOTE: low_watermark and high_watermark live in pbft_watermark_state_t
    // (single-owner rule per MEMORY §2.4). Access them via pbft_watermark_get().

    // V-set cache: VIEW-CHANGE messages received for the current view_change target
    uint8_t             vc_set_count;           // 0..n-1
    uint8_t             vc_set_from;            // single bitmask: bit i = VC received from node i (G19)
    const pbft_view_change_t* vc_set[7];        // pointers into RX message arena

    uint32_t            prepare_timeout_ms;     // default 1000
    uint32_t            commit_timeout_ms;      // default 1000
    uint32_t            viewchange_timeout_ms;  // default 2000
} pbft_view_state_t;
```

`vc_set[]` points into the static RX arena — no heap. When the arena is overwritten by a new RX, the pointers must be considered stale; to avoid that, **copy each VC into a fixed slot in `vc_set_arena[7]`** (each slot ≈ 200 B; 7 × 200 = 1.4 KB).

### 4.2 Prepared-set entry

```c
typedef struct __attribute__((packed)) {
    uint64_t   sequence;          // TX sequence number
    uint8_t    digest[32];        // SHA-256 of TX payload
} pbft_prepared_entry_t;
```

### 4.3 View-Change message (corrected format — fixes audit A1)

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  common header (msg_type=4, sender_id, reserved)               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                       view (uint32)                          |  ← current (suspected-faulty) view
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                     new_view (uint32)                         |  ← target view number
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |              checkpoint_seq (uint64)                          |  ← last stable checkpoint sequence
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |              checkpoint_digest (32 bytes)                     |  ← state digest at checkpoint
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |   n_prepared (uint8)   |      reserved (24 bits, MBZ)         |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  | { prepared_entry: n_prepared × (8 + 32) bytes }               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                  mac (32 bytes)                              |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  Size: 4 + 4 + 4 + 8 + 32 + 1 + 3 + n_prepared × 40 + 32
      = 88 + n_prepared × 40 bytes
```

**Empty prepared-set:** `n_prepared = 0`, total 88 B — fits ESP-NOW.
**Worst case (PBFT_VC_MAX_PREPARED=4):** `n_prepared = 4`, total = 88 + 160 = **248 B** — fits ESP-NOW 250 B with 2 B spare (H2 fix).

The MAC input is `view ‖ new_view ‖ checkpoint_seq ‖ checkpoint_digest ‖ n_prepared ‖ {prepared_entries}` — i.e. everything up to but excluding the MAC field itself.

### 4.4 New-View message (corrected format — fixes audit A2)

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  common header (msg_type=5, sender_id, reserved)               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                       view (uint32)                          |  ← new view number (v+1)
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |   num_vc (uint8) |  reserved (24 bits, MBZ)                   |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |              v_set_digest (32 bytes)                         |  ← SHA-256 of concatenated VC MACs
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  | { VC proof: num_vc × (4 + 4 + 4 + 8 + 32 + 1 + 3 + n × 40 + 32) }
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                  mac (32 bytes)                              |  ← over header + view + num_vc + v_set_digest + VC proofs
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  Then (no MAC trailer — these are independent PBFT messages):
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  | { O-set Pre-Prepare messages: independent Pre-Prepare pkts   }|  ← each carries its own Pre-Prepare MAC
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-++
```

**Size estimate (worst case n=7, n_prepared=4 each VC):**
- VC proof size = 88 + 4 × 40 = 248 B
- 5 VC proofs (≥ 2f+1) = 1240 B
- New-View outer = 4 + 4 + 1 + 3 + 32 + 1240 + 32 = 1316 B
- + O-set: typically 0-10 Pre-Prepares × 338 B = 0-3380 B
- **Grand total worst case ≈ 5.9 KB** → **UDP only.**

This is documented in PROTOCOL.md §8.2: because the New-View message exceeds the ESP-NOW 250 B frame, view-change as a feature **requires the build `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y`** and is sent over the single compiled-in `default_transport` (G11). There is no runtime fallback and no separate UDP transport; under `CONFIG_PBFT_TRANSPORT_ESP_NOW=y` view-change is not supported (PROTOCOL §8.1).

---

## 5. Algorithm — replica side

### 5.1 Pseudocode: handle VIEW-CHANGE

```c
void pbft_handle_view_change(const pbft_view_change_t* vc, uint8_t sender) {
    // 1. Validate sender and message
    if (sender >= PBFT_CLUSTER_SIZE) return;
    if (vc->new_view != view_state.view + 1) {
        log_warn("VC new_view=%lu != my view+1=%lu — stale or premature",
                 vc->new_view, view_state.view + 1);
        return;
    }
    // Verify MAC with hmac_keys[sender] (always — VC is never TOFU)
    if (pbft_verify_vc_mac(vc, sender) != ESP_OK) {
        log_warn("VC MAC invalid from %d", sender);
        return;
    }

    // 2. If I am the new primary, store the VC for V-set
    uint8_t new_primary = (uint8_t)((view_state.view + 1) % PBFT_CLUSTER_SIZE);
    if (new_primary == my_node_id) {
        if (!(view_state.vc_set_from & (1u << sender))) {
            view_state.vc_set_from |= (1u << sender);
            view_state.vc_set_count++;
            // Copy VC into arena slot
            memcpy(&vc_set_arena[sender], vc, pbft_vc_size(vc));
            view_state.vc_set[sender] = &vc_set_arena[sender];
        }
        // Trigger when quorum reached
        if (view_state.vc_set_count >= PBFT_QUORUM &&
            view_state.phase != PBFT_VIEW_PHASE_WAIT_NEW) {
            view_state.phase = PBFT_VIEW_PHASE_WAIT_NEW;
            view_state.new_view_started_ms = now_ms();
            pbft_send_new_view();   // see §5.3
        }
        return;
    }

    // 3. Replica side: record VC toward quorum
    if (!(view_state.vc_set_from & (1u << sender))) {
        view_state.vc_set_from |= (1u << sender);
        view_state.vc_set_count++;
        memcpy(&vc_set_arena[sender], vc, pbft_vc_size(vc));
        view_state.vc_set[sender] = &vc_set_arena[sender];
    }

    // 4. If 2f+1 received and I have not yet sent VC, send now (PBFT §4.2)
    if (view_state.vc_set_count >= PBFT_QUORUM &&
        view_state.phase == PBFT_VIEW_PHASE_NORMAL) {
        view_state.phase = PBFT_VIEW_PHASE_CHANGING;
        view_state.view_change_started_ms = now_ms();
        pbft_send_view_change(view_state.view + 1);
    }
}
```

### 5.2 Pseudocode: send VIEW-CHANGE

```c
void pbft_send_view_change(uint32_t new_view) {
    // Build prepared-set from PBFT log: every entry in PREPARED or COMMITTED state
    // with sequence > checkpoint_seq.
    uint8_t n = 0;
    // CRITICAL: do NOT allocate on stack — pbft_prepared_entry_t × 100 = 4 KB,
    // and o_set (oops) × 100 = ~33 KB, both would exceed the 8 KB FreeRTOS
    // task stack. Use a static scratch arena (BSS) instead, sized to the
    // actual PBFT_VC_MAX_PREPARED=4 cap.
    static pbft_prepared_entry_t prepared[PBFT_VC_MAX_PREPARED];

    for (int i = 0; i < PBFT_LOG_MAX_ENTRIES; i++) {
        pbft_log_entry_t* e = &pbft_log[i];
        if (e->state == PBFT_STATE_FREE) continue;
        if (e->sequence <= pbft_watermark_get_low()) continue;
        if (e->state != PBFT_STATE_PREPARED &&
            e->state != PBFT_STATE_COMMITTED &&
            e->state != PBFT_STATE_EXECUTED) continue;
        prepared[n].sequence = e->sequence;
        memcpy(prepared[n].digest, e->digest, 32);
        n++;
    }

    // Build VIEW-CHANGE
    pbft_view_change_t vc = {
        .view            = view_state.view,
        .new_view        = new_view,
        .checkpoint_seq  = pbft_watermark_get_low(),
        .n_prepared      = n,
    };
    // Fill checkpoint_digest: SHA-256 of app state at low_watermark (TBD; app-supplied)
    pbft_checkpoint_digest_at(pbft_watermark_get_low(), vc.checkpoint_digest);
    // Fill prepared entries (variable length — pack into a buffer)
    memcpy(((uint8_t*)&vc) + offsetof(pbft_view_change_t, prepared),
           prepared, n * sizeof(pbft_prepared_entry_t));

    // Compute MAC over the entire VC header + body
    size_t vc_size = pbft_vc_size(&vc);
    pbft_compute_mac(my_node_id, view_state.view, /*sequence=*/0, PBFT_MSG_VIEW_CHANGE,
                     /*digest=*/vc.checkpoint_digest,
                     (uint8_t*)&vc + 4, vc_size - 4 - 32,  // everything except common header and trailing MAC
                     ((uint8_t*)&vc) + vc_size - 32);

    // No runtime fallback. Transport chosen at compile time.
    // If CONFIG_PBFT_TRANSPORT_ESP_NOW=y and vc_size > 250, this returns
    // PBFT_NET_ERR_INVALID from espnow_broadcast() (defensive runtime check;
    // the _Static_assert in MEMORY.md should catch this at compile time
    // when PBFT_VC_MAX_PREPARED is large).
    (void)default_transport->broadcast(&vc, vc_size);

    // Local state
    view_state.phase = PBFT_VIEW_PHASE_CHANGING;
    view_state.view_change_started_ms = now_ms();
    // Retain: log entries, low/high watermarks (do NOT reset — see PBFT §4.2)
}
```

**Critical:** the prepared-set must include **EXECUTED** entries (already-committed TXs) in addition to PREPARED. Otherwise the new primary cannot prove that already-committed TXs will not be re-executed.

### 5.3 Pseudocode: handle NEW-VIEW (replica)

```c
void pbft_handle_new_view(const pbft_new_view_t* nv, uint8_t sender) {
    // 1. Validate sender is the new primary.
    //    The view has NOT advanced yet at validation time, so the new primary
    //    is (view + 1) % n, not view % n (G15 off-by-one fix).
    uint8_t expected_primary = (uint8_t)((view_state.view + 1) % PBFT_CLUSTER_SIZE);
    if (sender != expected_primary) {
        log_warn("New-View from non-primary %d (expected %d)",
                 sender, expected_primary);
        return;
    }

    // 2. Verify outer MAC
    if (pbft_verify_nv_outer_mac(nv, sender) != ESP_OK) {
        log_warn("New-View outer MAC invalid");
        return;
    }

    // 3. Reconstruct V-set from embedded VC proofs
    if (nv->num_vc < PBFT_QUORUM) {
        log_warn("New-View has only %d VCs (need >= %d)", nv->num_vc, PBFT_QUORUM);
        return;
    }

    // 4. For each embedded VC, verify its MAC and that its sender is in the set
    uint8_t distinct_senders = 0;   // single bitmask: bit i = VC from node i (G19)
    for (int i = 0; i < nv->num_vc; i++) {
        const pbft_view_change_t* vc = pbft_nv_get_vc(nv, i);
        if (vc->new_view != nv->view) { log_warn("VC new_view mismatch"); return; }
        if (vc->sender_id >= PBFT_CLUSTER_SIZE) return;
        if (distinct_senders & (1u << vc->sender_id)) { log_warn("Duplicate VC sender"); return; }
        distinct_senders |= (1u << vc->sender_id);
        if (pbft_verify_vc_mac(vc, vc->sender_id) != ESP_OK) {
            log_warn("Embedded VC MAC invalid"); return;
        }
    }
    if (pbft_count_set_bits(distinct_senders) < PBFT_QUORUM) {
        log_warn("New-View V-set lacks quorum of distinct senders");
        return;
    }

    // 5. Recompute v_set_digest and compare
    uint8_t expected_digest[32];
    pbft_compute_v_set_digest(nv, expected_digest);
    if (memcmp(expected_digest, nv->v_set_digest, 32) != 0) {
        log_warn("New-View v_set_digest mismatch");
        return;
    }

    // 6. Compute O-set locally (must match what new primary computed).
    // CRITICAL: o_set must NOT be on stack — pbft_pre_prepare_t × 100 = ~33 KB
    // would crash the 8 KB FreeRTOS task stack. Use a static scratch arena
    // (BSS) sized to the documented cap (PBFT_NVIEW_O_SET_MAX = 32 entries
    // = ~10.8 KB). Caller is single-threaded (one VIEW-CHANGE at a time per
    // replica), so a single scratch slot is safe.
    static pbft_pre_prepare_t o_set_arena[PBFT_NVIEW_O_SET_MAX];
    pbft_pre_prepare_t* o_set = o_set_arena;
    uint8_t n_o = pbft_compute_o_set(nv, o_set);   // deterministic, see §6
    if (n_o == PBFT_O_SET_NEEDS_CATCHUP) {
        // Too far behind to replay the O-set in place (G15 overflow guard).
        // pbft_compute_o_set already issued a state-transfer request; abort the
        // New-View apply and retry verification once caught up.
        log_warn("New-View: node behind O-set span — catching up via state-transfer");
        return;
    }
    // Compare with embedded Pre-Prepares
    if (n_o != nv->n_pre_prepares) {
        log_warn("O-set length mismatch (local=%d, sent=%d)", n_o, nv->n_pre_prepares);
        return;
    }
    for (int i = 0; i < n_o; i++) {
        const pbft_pre_prepare_t* sent = pbft_nv_get_pp(nv, i);
        if (sent->sequence != o_set[i].sequence ||
            memcmp(sent->digest, o_set[i].digest, 32) != 0) {
            log_warn("O-set entry %d mismatch", i);
            return;
        }
        // Verify Pre-Prepare MAC against sender (the new primary)
        if (pbft_verify_pp_mac(sent, expected_primary) != ESP_OK) {
            log_warn("O-set Pre-Prepare MAC invalid");
            return;
        }
    }

    // 7. All checks passed — accept the new view
    view_state.view = nv->view;
    view_state.primary_id = expected_primary;
    view_state.phase = PBFT_VIEW_PHASE_NORMAL;
    view_state.vc_set_count = 0;
    view_state.vc_set_from = 0;

    // 8. For each O-set Pre-Prepare, run it through the normal Pre-Prepare handler
    for (int i = 0; i < n_o; i++) {
        pbft_handle_pre_prepare(&o_set[i], expected_primary);
    }
}
```

---

## 6. Algorithm — new primary side

### 6.1 O-set construction (the core of view-change)

The new primary computes a deterministic O-set from the union of the prepared-sets in the V-set. PBFT §4.4 gives the algorithm. Adapted for esp-pbft. On success the function returns the O-set length (`0..PBFT_LOG_MAX_ENTRIES`); the reserved sentinel `PBFT_O_SET_NEEDS_CATCHUP` (`0xFF`) signals that this node is too far behind and must state-transfer first (G15):

```c
uint8_t pbft_compute_o_set(const pbft_new_view_t* nv,
                            pbft_pre_prepare_t* out) {
    typedef struct { uint64_t seq; uint8_t digest[32]; uint8_t n_votes; } prepared_with_count_t;

    prepared_with_count_t candidates[PBFT_LOG_MAX_ENTRIES];
    uint8_t n_cand = 0;

    // Step 1: collect every (seq, digest) pair across all VCs, counting appearances
    for (int i = 0; i < nv->num_vc; i++) {
        const pbft_view_change_t* vc = pbft_nv_get_vc(nv, i);
        for (int j = 0; j < vc->n_prepared; j++) {
            const pbft_prepared_entry_t* pe = &vc->prepared[j];
            // Find or create candidate
            prepared_with_count_t* c = NULL;
            for (int k = 0; k < n_cand; k++) {
                if (candidates[k].seq == pe->sequence &&
                    memcmp(candidates[k].digest, pe->digest, 32) == 0) {
                    c = &candidates[k];
                    break;
                }
            }
            if (!c) {
                c = &candidates[n_cand++];
                c->seq = pe->sequence;
                memcpy(c->digest, pe->digest, 32);
                c->n_votes = 0;
            }
            c->n_votes++;
        }
    }

    // Step 2: filter — keep only entries with seq > low_watermark_max
    //         where low_watermark_max = max checkpoint_seq across VCs
    uint64_t lw_max = 0;
    for (int i = 0; i < nv->num_vc; i++) {
        const pbft_view_change_t* vc = pbft_nv_get_vc(nv, i);
        if (vc->checkpoint_seq > lw_max) lw_max = vc->checkpoint_seq;
    }

    // Step 2b: compute the O-set UPPER BOUND deterministically from the V-set
    //   (G15). The bound is max_prepared_seq = the maximum sequence appearing in
    //   ANY prepared-set across the V-set — identical on every node. It MUST NOT
    //   be view_state.next_sequence (a node-local counter that differs per node
    //   and would make the O-set non-deterministic, breaking verification).
    uint64_t max_prepared_seq = lw_max;   // floor: nothing below the checkpoint
    for (int k = 0; k < n_cand; k++) {
        if (candidates[k].seq > max_prepared_seq) max_prepared_seq = candidates[k].seq;
    }

    // Step 2c: overflow guard (G15). The o_set[] array (and the local log) hold at
    //   most PBFT_LOG_MAX_ENTRIES entries. If the span from the checkpoint to the
    //   highest prepared sequence exceeds that capacity, this node is too far
    //   behind to replay the O-set in place: the New-View is invalid for it and it
    //   MUST catch up via state-transfer (CHECKPOINT.md §8) first, then re-validate.
    //   Never silently overflow o_set[PBFT_LOG_MAX_ENTRIES].
    if (max_prepared_seq - lw_max > PBFT_LOG_MAX_ENTRIES) {
        log_warn("O-set span %llu exceeds log capacity %d — catch up via "
                 "state-transfer before applying New-View",
                 max_prepared_seq - lw_max, PBFT_LOG_MAX_ENTRIES);
        pbft_request_state_transfer(lw_max + 1);   // CHECKPOINT.md §8
        return PBFT_O_SET_NEEDS_CATCHUP;            // caller aborts New-View apply
    }

    // Step 3: for each candidate, decide:
    //   - seq > lw_max AND n_votes >= f+1 (≥3): re-propose with this digest
    //     (≥ f+1 honest replicas prepared it, so the digest is canonical)
    //   - seq <= lw_max: drop (already in a stable checkpoint)
    //   - n_votes < f+1 OR no agreement on digest: assign a no-op (NULL digest)
    uint8_t n_o = 0;
    for (uint64_t seq = lw_max + 1;
         seq <= max_prepared_seq;
         seq++) {
        // Find candidate for this seq
        prepared_with_count_t* c = NULL;
        for (int k = 0; k < n_cand; k++) {
            if (candidates[k].seq == seq) { c = &candidates[k]; break; }
        }

        pbft_pre_prepare_t* pp = &out[n_o++];
        pp->view = nv->view;
        pp->sequence = seq;
        pp->primary_id = my_node_id;
        pp->payload_len = 0;

        if (c && c->n_votes >= PBFT_F + 1) {
            memcpy(pp->digest, c->digest, 32);
            // Try to recover the payload from local log
            pbft_log_entry_t* e = pbft_log_find(view_state.view, seq);
            if (e && e->payload_len > 0) {
                pp->payload = e->payload;
                pp->payload_len = e->payload_len;
            } else {
                // Payload missing — fall back to no-op to preserve safety
                log_warn("O-set: payload missing for seq %llu — no-op", seq);
                pp->payload_len = 0;
                memset(pp->digest, 0, 32);
            }
        } else {
            // No agreement OR seq not in any prepared-set — no-op (skip)
            log_info("O-set: no-op for seq %llu", seq);
            pp->payload_len = 0;
            memset(pp->digest, 0, 32);
        }
    }

    return n_o;
}
```

**Why this is deterministic:** every honest replica that receives the V-set will compute the identical O-set (same algorithm, same inputs, same f+1 threshold). Replicas then verify the new-primary's New-View by recomputing the O-set and comparing entry-by-entry.

### 6.2 Pseudocode: send NEW-VIEW

```c
void pbft_send_new_view(void) {
    // 1. Compute O-set deterministically from V-set (cache).
    // CRITICAL: not on stack — see warning in pbft_handle_new_view().
    static pbft_pre_prepare_t o_set_arena[PBFT_NVIEW_O_SET_MAX];
    pbft_pre_prepare_t* o_set = o_set_arena;
    uint8_t n_o = pbft_compute_o_set_from_cache(o_set);

    // 2. Build New-View message
    //    Layout: header + view + num_vc + v_set_digest + VC proofs + outer MAC
    //            + (separately) O-set Pre-Prepares (each with own MAC)
    size_t vc_total = 0;
    for (int i = 0; i < view_state.vc_set_count; i++) {
        vc_total += pbft_vc_size(view_state.vc_set[i]);
    }
    size_t nv_size = 4 + 4 + 1 + 3 + 32 + vc_total + 32;
    uint8_t* nv_buf = (uint8_t*)pbft_tx_buffer;  // static 4 KB TX buffer
    // ... fill nv_buf ...

    // 3. Compute v_set_digest = SHA-256(concat of all VC MACs)
    pbft_compute_v_set_digest_from_cache(nv_buf + offset_v_set_digest);

    // 4. Compute outer MAC
    pbft_compute_mac(my_node_id, nv->view, /*sequence=*/0, PBFT_MSG_NEW_VIEW,
                     nv_buf + offset_v_set_digest, /*digest arg*/
                     nv_buf + 4, nv_size - 4 - 32,
                     nv_buf + nv_size - 32);

    // 5. Append O-set Pre-Prepares (each with own MAC computed at prep time)
    //    These are full Pre-Prepare messages (each ~80 + payload B)
    //    They go in the SAME UDP packet or in a continuation if too big.

    // 6. Send via the single compiled-in transport (G11). View-change is only
    //    supported when built with CONFIG_PBFT_TRANSPORT_WIFI_UDP=y, because the
    //    New-View message (~5.9 KB worst case) far exceeds the ESP-NOW 250 B frame.
    //    There is NO runtime fallback and NO separate udp_transport: the build
    //    that enables view-change already has default_transport == the UDP transport.
    //    Under CONFIG_PBFT_TRANSPORT_ESP_NOW=y view-change is not compiled in
    //    (see PROTOCOL §8.1), so this path is unreachable in that build.
#if !defined(CONFIG_PBFT_TRANSPORT_WIFI_UDP)
#error "View-change requires CONFIG_PBFT_TRANSPORT_WIFI_UDP=y (New-View exceeds ESP-NOW frame)"
#endif
    default_transport->broadcast(nv_buf, total_size);
}
```

---

## 7. Safety argument (sketch)

The standard PBFT safety argument (PBFT paper §4.4) applies. Key invariants:

1. **Prepared invariant.** If an honest replica `r` has `(seq, d)` prepared in view `v`, then no honest replica commits any `(seq, d')` with `d' ≠ d` in any view `v' ≤ v`.

   *Proof sketch:* `r` prepared `(seq, d)` requires a Pre-Prepare from the primary of view `v` AND `≥ 2f+1` matching Prepares. At most `f` of those Prepares can be Byzantine; so `≥ f+1` honest replicas also have `(seq, d)` prepared. To commit a conflicting `(seq, d')` at the same seq, an attacker would need `2f+1` Prepares for `d'`; but only `f` replicas can be Byzantine, so the `f+1` honest replicas that prepared `d` will refuse to Prepare `d'`. Contradiction.

2. **O-set correctness.** The O-set is computed deterministically from the V-set, and every honest replica recomputes it identically. The new primary cannot include a Pre-Prepare that disagrees with what the replicas will compute (else verification fails in §5.3 step 6).

3. **No committed-but-lost.** Any TX that reached COMMITTED or EXECUTED in view `v` appears in the prepared-set of every honest replica (because the committed-state transition requires `2f+1` matching Commits, of which `f+1` are honest). Those `f+1` honest replicas include the TX in their prepared-set; the new primary's O-set computation in §6.1 includes the entry with `n_votes ≥ f+1`. Hence the new primary re-proposes the same `(seq, digest)`. Replicas that already executed it can detect this and skip re-execution (compare with their `last_executed_seq`).

---

## 8. Liveness argument (sketch)

1. **Eventually some replica triggers.** If the primary is faulty (no Pre-Prepare after `submit()`), every replica's `submit()`-side timeout (or any in-flight Prepare/Commit timeout) fires within `prepare_timeout_ms` (default 1000 ms). Each replica sends a VIEW-CHANGE.

2. **Quorum of VIEW-CHANGEs reached.** With at most `f=2` Byzantine replicas, at least `n - f = 5` honest replicas send VIEW-CHANGEs within the same window. The first `2f+1 = 5` to arrive at any replica (including the new primary) trigger state transition to `WAIT_NEW`.

3. **New primary computes and broadcasts.** The new primary (`view+1 mod n`) is honest (probability `(n-f)/n = 5/7` per round; with retries, 1 in O(log rounds)). It computes O-set and broadcasts NEW-VIEW.

4. **New-View reaches all honest replicas.** UDP broadcast is unreliable, but the new primary can retransmit until it has received `2f+1` ACKs (replicas reply with Prepare for at least one O-set entry). Or simpler: replicas that don't receive New-View within `viewchange_timeout_ms` send `VIEW-CHANGE(v+2)` → process repeats with the next primary.

5. **Progress.** Within `O(viewchange_timeout_ms)` (default 2000 ms), the cluster is back in `VIEW_NORMAL` for view `v+1`. Throughput is briefly reduced; steady-state latency resumes.

**Worst-case liveness delay:** `prepare_timeout_ms + viewchange_timeout_ms × (1 + ε)` per view-change, where ε is the cascade probability. Default ≈ 3 seconds per view-change. View-changes are rare in practice (only on primary failure).

---

## 9. Edge cases

### 9.1 Replica crashes during view-change

A replica that sent VIEW-CHANGE then crashes before the new view begins: when it reboots, its current `view_state.view` is stale (NVS-persisted). On reboot, it must:

1. Re-do Pattern Y handshake (TRNG keypair + Hello) — peers re-derive HMAC keys (soft re-handshake).
2. Read `view_state.view` and `view_state.next_sequence` from NVS.
3. Wait for any incoming Pre-Prepare. If none within `viewchange_timeout_ms`, assume it is still mid-view-change and send its own `VIEW-CHANGE(view+1)` based on its stored state.
4. Eventually catch up via New-View + O-set.

This is correct but slow. v2: include `last_stable_checkpoint` in NVS to skip ahead.

### 9.2 Primary recovery (former faulty primary comes back)

If the former primary rejoins the cluster:
- It boots fresh (TRNG keypair) — old keypair is gone.
- It broadcasts Hello — peers see a new pubkey for `sender_id = old_primary_id`.
- Peers do **not** trust this automatically (cached pubkey mismatch). Two options:
  - (a) Reject with alarm (forces operator intervention).
  - (b) Accept as a re-handshake (matches Y-5 behavior).
- Recommended: (b), with a log warning. The primary's voting weight is not restored until it completes the Y-5-style re-handshake AND its `view` catches up via New-View.

### 9.3 Network partition — minority side

A partition that contains `< 2f+1 = 5` replicas (e.g., 3 honest + 2 Byzantine) cannot drive view-change: no quorum of VIEW-CHANGEs. They sit in `VIEW_NORMAL` waiting for Pre-Prepares that never arrive. After `viewchange_timeout_ms`, they escalate to `VIEW-CHANGE(view+2)` → still no quorum → infinite escalation. **Liveness halts on this partition until it heals.**

Mitigation: each replica stops escalating after `view_state.view - boot_view > MAX_VIEW_GAP` (e.g., 100) and switches to "partition mode" (no consensus, listen-only). On hearing a New-View, it rejoins.

### 9.4 Network partition — majority side

The `≥ 2f+1 = 5` side has quorum; they proceed with view-change → new primary → continue. When the partition heals, the minority re-bootstraps (read NVS, do soft re-handshake, accept incoming New-View, catch up via O-set).

### 9.5 Cascading view-changes

A replica can be in view-change round 3 (target view v+3) while another replica is still in view-change round 1 (target view v+1). The new primary for v+1 may not exist yet (no quorum of VCs targeting v+1). The algorithm must accept "view-change to v+3" as evidence that "view-change to v+1" should also proceed.

**Rule:** on receiving `VIEW-CHANGE(new_view = v+k)` for `k > 1`, accept it as a vote for all intermediate targets `v+1, v+2, ..., v+k`. The first to reach quorum (whichever target) drives the next transition.

This is captured in §5.1 step 1 by allowing any `new_view >= view_state.view + 1`.

### 9.6 Replica receives VIEW-CHANGE for a view lower than its current view

Drop silently (stale message). Example: replica just completed view-change to v+1; a late VIEW-CHANGE for v+1 arrives; replica is already past it.

### 9.7 New primary fails before sending NEW-VIEW

New primary crashes mid-view-change. The `viewchange_timeout_ms` fires on all replicas → they send `VIEW-CHANGE(v+2)` → new new primary takes over. Standard PBFT §4.4 case.

---

## 10. Memory and wire-cost summary

| Item | Static size | Notes |
|------|-------------|-------|
| `pbft_view_state_t` | ~120 B | single instance |
| `vc_set_arena[7]` (each VC slot) | 7 × 248 B = 1.7 KB | PBFT_VC_MAX_PREPARED=4 (H2 fix); in practice all slots are ≤ 248 B. Historical upper bound of 4088 B is unreachable with the current Kconfig range cap. |
| Practical static cost | ~1.8 KB | aligns with HANDOVER §3.7 "View-change state ~1 KB" + a few hundred B slack |
| View-Change wire (typical) | 88 + n_prepared × 40 B | n_prepared ≈ 0-4 → 88-248 B (fits ESP-NOW 250 B) |
| View-Change wire (worst) | 248 B | n_prepared = 4 (= PBFT_VC_MAX_PREPARED); fits ESP-NOW |
| New-View wire (typical) | 2500-5000 B | UDP only |
| New-View wire (worst) | ~5.9 KB | UDP only |

**Conclusion:** View-Change fits ESP-NOW in typical cases (≤ 250 B); New-View (~5.9 KB worst case) requires a `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y` build and is sent via `default_transport` (G11).

---

## 11. Configuration (Kconfig additions)

```kconfig
# sdkconfig.defaults additions
CONFIG_PBFT_VIEWCHANGE_TIMEOUT_MS=2000
CONFIG_PBFT_PREPARE_TIMEOUT_MS=1000
CONFIG_PBFT_COMMIT_TIMEOUT_MS=1000
CONFIG_PBFT_MAX_VIEW_GAP=100        # before declaring "partition mode"
CONFIG_PBFT_VIEWCHANGE_RETRANSMIT=3 # VC/NEW-VIEW retransmit count
CONFIG_PBFT_VC_MAX_PREPARED=4       # cap on n_prepared in VC (enforces ≤ 250 B ESP-NOW; see "no-runtime-fallback")
```

`PBFT_VC_MAX_PREPARED` is the most important: it bounds the View-Change wire size to `88 + 4 × 40 = 248 B` — fits ESP-NOW 250 B with 2 B spare. Any prepared entries beyond the cap are aggregated (we send only the highest-sequence `PBFT_VC_MAX_PREPARED` entries; lower ones will be re-proposed by the new primary from the local log anyway, because every replica keeps the log across view-change). If you need more than 4 prepared entries carried, you MUST select `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y` and route View-Change through UDP — see PROTOCOL.md §2.5 / §8 for the no-runtime-fallback rule.

---

## 12. Open questions (view-change level)

| # | Question | Recommendation |
|---|----------|----------------|
| V1 | What if O-set Pre-Prepare payload is missing from local log (replica rebooted between prepare and view-change)? | No-op (skip) that sequence — see §6.1 step 3 fallback. |
| V2 | Should New-View include a checkpoint proof, or just checkpoint_seq? | checkpoint_seq + checkpoint_digest (32 B) is enough; replicas verify against their own checkpoint. |
| V3 | Cascade limit? | `PBFT_MAX_VIEW_GAP = 100` then "partition mode". |
| V4 | Should new primary broadcast New-View multiple times for reliability? | Yes, retransmit up to `PBFT_VIEWCHANGE_RETRANSMIT = 3` times with jittered backoff. |
| V5 | Should VIEW-CHANGE be sent only on T1-T4 or also on T5 (timeout)? | Both — T5 escalates to v+2. |

---

## 13. References

- [HANDOVER.md](./HANDOVER.md) — n=7, f=2, quorum=5
- [ARCHITECTURE.md](./ARCHITECTURE.md) §7 — preview
- [CONSENSUS.md](./CONSENSUS.md) — state machine that view-change integrates with
- [CRYPTO.md](./CRYPTO.md) — HMAC for VC and New-View MACs
- [PROTOCOL.md](./PROTOCOL.md) — wire format (§6.5, §6.6 corrected here)
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — A1, A2 (now fixed), A6 (related to log/watermark)
- Castro & Liskov, "Practical Byzantine Fault Tolerance" (OSDI 1999), §4 — view-change protocol

---

**End of VIEW-CHANGE.md (v0.1 draft — addresses audit issues A1, A2)**