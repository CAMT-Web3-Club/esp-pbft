# esp-pbft — Design Audit (2026-06-27)

> **Scope:** cross-doc consistency, gaps, ambiguity, safety/liveness risks, ESP32-C3 feasibility.
> **Method:** read every existing doc (HANDOVER, ARCHITECTURE, CONSENSUS, CRYPTO, PROTOCOL, CODING_STANDARD), compare against the PBFT paper (Castro & Liskov, OSDI 1999) and ESP-IDF v6.0.1 PSA/ESP-NOW realities.
> **Verdict:** design is **fundamentally sound** (Pattern Y + n=7, f=2 + 3-phase + Y-5 re-gen + static memory) but has **~30 specific issues** that must be fixed before implementation. Most are local; a few are protocol-breaking.

Severity legend: 🔴 protocol-breaking · 🟠 behavior-changing · 🟡 documentation/clarity · 🟢 nit.

---

## A. Protocol-breaking issues (must fix)

### 🔴 A1. View-Change wire format omits the prepared-set
**File:** PROTOCOL.md §6.5
**Problem:** View-Change carries `(view, new_view, checkpoint_seq, checkpoint_digest, mac)` only. Standard PBFT requires each replica to include the **prepared set** — the list of `(seq, digest)` pairs for every TX the replica has in PREPARED or COMMITTED state above the checkpoint. Without it, the new primary cannot construct a defensible O-set (Pre-Prepares to re-propose), and replicas cannot verify the new-view.
**Fix:** add `n_prepared : uint8` + `n_prepared × {seq:u64, digest:32}` to the View-Change body. With PBFT_LOG_MAX_ENTRIES=100 and worst case `2f+1=5` VCs in a New-View, total size grows but still ≤ 4 KB (superseded R2: ~5.9 KB worst case) — fits UDP, exceeds ESP-NOW (split into multiple ESP-NOW frames or force UDP for New-View, which is already the plan).

### 🔴 A2. New-View format conflates V-set proof with O-set delivery
**File:** PROTOCOL.md §6.6
**Problem:** `New-View = view | num_vc | {VC proofs} | {Pre-Prepare, var} | mac` is incomplete. The O-set must be derived deterministically by the new primary from the union of the prepared-sets in the V-set (PBFT §4.4 — "the new-view message contains the set of pre-prepares that need to be re-proposed"). Sending the Pre-Prepares inline is fine, but the mac should be over each Pre-Prepare individually (each carries its own Pre-Prepare MAC), and the New-View itself needs a separate MAC that binds `view` + `num_vc` + a digest of the V-set.
**Fix:** split MACs — (a) each embedded Pre-Prepare carries its own HMAC; (b) New-View outer MAC binds `view ‖ num_vc ‖ SHA-256(concat of all VC MACs)`. New-View ≥ 600 B; UDP only.

### 🔴 A3. `pbft_submit` rejects all non-primary callers → single point of failure
**File:** CONSENSUS.md §5.2
**Problem:** `if (my_node_id != view_state.primary_id) return PBFT_ERR_NOT_PRIMARY;` makes the primary a single point of failure for submission. Standard PBFT allows ANY replica to accept a client request and forward it (via `<REQUEST, o, t, c>` relaying). With this restriction, if the primary is alive but slow or partitioned in a way that bypasses view-change detection, no submissions commit.
**Fix:** keep the API simple for v1 by **defining the application as the "client"** and require the app to track the primary (it gets `view` and `primary_id` via `pbft_get_state()`). Document explicitly: "app must re-submit to the current primary on each view change." v2 can add automatic forwarding.

### 🔴 A4. `psa_key_agreement` pseudocode won't compile
**File:** CRYPTO.md §4.1
**Problem:** `psa_key_agreement(PSA_ALG_ECDH, my_priv_key_id, peer_pub_keys[peer], …)` passes raw 64-byte pubkey bytes. PSA's `psa_key_agreement` accepts only a `psa_key_id_t` for the public side. To pass raw bytes you must use `psa_raw_key_agreement()` (deprecated path) or import the peer's pubkey into a transient PSA key first via `psa_import_key()` with `PSA_KEY_TYPE_ECC_PUBLIC_KEY(PSA_ECC_FAMILY_SECP_R1)`. The current pseudocode compiles to a type error.
**Fix:** add a `peer_pub_key_ids[7]` static array of `psa_key_id_t`; on Hello-receive, `psa_destroy_key` the old ID (if any), then `psa_import_key` the new pubkey. Then `psa_key_agreement(PSA_ALG_ECDH, my_priv, peer_pub_key_ids[peer], …)`. Also: `psa_destroy_key` these IDs on every Y-5 re-gen of peer.

### 🔴 A5. Memory budget contradiction across docs
**Files:** HANDOVER §3.7 line "PBFT log 100 × 60 B = ~6 KB"; ARCHITECTURE §11 "100 × ~120 B = ~12 KB"; CONSENSUS §13 "~120 B per entry".
**Problem:** the per-entry size depends on whether the 256-byte payload is stored inline (≈ 280 B/entry) (superseded R2: ~360 B/entry) or via a pointer (≈ 96 B/entry but payload memory comes from elsewhere). Neither figure matches the "60 B" claim in HANDOVER. The HANDOVER total "40 KB" therefore doesn't reconcile.
**Fix:** define `pbft_log_entry_t` with **inline** payload (one struct = ~280 B (superseded R2: ~360 B), 100 entries = ~28 KB (superseded R2: ~36 KB)) since the design rules out heap. Update HANDOVER §3.7, ARCHITECTURE §11, CONSENSUS §13 to match. Then the static total is ~55 KB for ESP-NOW build (superseded R2: ~72 KB esp-pbft / ~117 KB grand) (still within 400 KB SRAM but tighter than originally advertised). MEMORY.md to provide full layout with `static_assert` catalog.

### 🔴 A6. No high-watermark advancement → primary can stall after log fills
**File:** CONSENSUS.md §8.4
**Problem:** `pbft_log_full()` returns true when `next_sequence > high_watermark + PBFT_LOG_MAX_ENTRIES`. But **who advances `high_watermark`?** Not specified. Without it, once the log fills, the primary must stop assigning sequences → liveness halt.
**Fix:** in CHECKPOINT.md (to be written): `high_watermark := low_watermark + PBFT_LOG_MAX_ENTRIES` is computed at boot; on every stable checkpoint `low_watermark := checkpoint_seq`, then `high_watermark := low_watermark + PBFT_LOG_MAX_ENTRIES`. Document this in CHECKPOINT.md §4.

### 🔴 A7. Anti-replay breaks across view-change unless sequences are view-global
**File:** CRYPTO.md §5.2
**Problem:** MAC binds `(view, sequence)`. Standard PBFT keeps sequences monotonic across views (`high_watermark + 1` is the first seq in view v+1), so the anti-replay works. The design says sequences are "monotonic" but does not state this explicitly.
**Fix:** add to CONSENSUS.md: "Sequence numbers are view-global and monotonic across view-changes. The first Pre-Prepare in view v+1 carries sequence `low_watermark + 1`." (Already implied by PBFT but should be explicit.)

---

## B. Behavior-changing issues (should fix)

### 🟠 B1. TOFU accepts any first Hello per channel — vulnerable to MITM at boot
**File:** CRYPTO.md §6.2, §10.3
**Problem:** "First Hello from peer is trusted" — if an attacker controls the RF channel at boot, they can substitute any node's pubkey before the legitimate node's Hello arrives. The legitimate Hello then fails the "must match cached pubkey" check and is rejected. All future messages from the legitimate node are dropped. The cluster becomes effectively attacker-controlled.
**Fix:** add physical-security assumption explicitly to the threat model; consider (for v2) signing the Hello with a manufacturer-injected long-term key (DS peripheral on ESP32-C3 supports RSA-2048, but not ECDSA — would need RSA-PSS). For v1: documented assumption only.

### 🟠 B2. PSA crypto allocates heap internally → "no heap" claim is misleading
**File:** ARCHITECTURE.md §11
**Problem:** §11.5 admits PSA allocates internally, but the top-line "static memory only" message implies otherwise. Compilers with `-fno-malloc-dce` (set by ESP-IDF) don't help here — PSA's `psa_mac_compute` uses a transient internal buffer.
**Fix:** restate: "esp-pbft source contains zero `malloc/free`. PSA crypto and FreeRTOS allocate internally; their peaks are bounded and reproducible." Add a heap-peak measurement to TEST-PLAN.md.

### 🟠 B3. Hello MAC uses zero MAC for first Hello → wire-layer only authentication
**File:** CRYPTO.md §6.1
**Problem:** "MAC may be zeros (peer has no key yet) — accept." This means the receiver has no way to verify the sender actually sent the message (anyone can forge an all-zero MAC). Combined with TOFU, the trust anchor is: (a) attacker is not present during boot, OR (b) attacker can't predict which channel the legitimate node will transmit on.
**Fix:** for re-handshake (Y-5) the MAC is mandatory (peer already has the key). For initial Hello, document the trust assumption. Add to threat model.

### 🟠 B4. PSA key attributes for ECDSA-keypair used only for ECDH should disable signing
**File:** CRYPTO.md §3.3
**Problem:** Sets `PSA_KEY_USAGE_DERIVE`. Correct. But doesn't set `PSA_KEY_USAGE_SIGN_HASH` → undefined behavior is prevented. Should also set `PSA_KEY_USAGE_VERIFY_HASH`? No — only DERIVE. The current code is correct, but a future maintainer might add SIGN without realizing the security implication (a stolen DERIVE-only key can't sign, but a stolen SIGN-capable key can impersonate). The KDF-then-MAC pattern protects against this.
**Fix:** add a `// NOTE:` comment in the attributes setup explicitly: "NEVER add `PSA_KEY_USAGE_SIGN_HASH` here. A signing key on every node enables ECDSA-spoof at runtime if compromised."

### 🟠 B5. Per-entry bitmask `uint8_t[7]` wastes 7 bits; per-bit bitmask uint8 doesn't match cluster of 7
**File:** CONSENSUS.md §3.1 (struct), §7.2 (algorithm)
**Problem:** `prepare_from[7]` is an array of 7 bitmasks (one per replica), not a single bitmask. The quorum check `pbft_count_set_bits(bitmask) >= 5` assumes ONE bitmask, but the struct holds 7 separate ones. Inconsistency in pseudocode.
**Fix:** pick one representation. Recommendation: per-entry `uint8_t prepare_bitmask` (one byte, bit i = "replica i sent Prepare"). `count_set_bits` is then `__builtin_popcount(entry->prepare_bitmask)`. Reduces struct size by 6 bytes per entry (×100 = 600 B saved).

### 🟠 B6. `state_entered_ms` referenced but not in struct
**File:** CONSENSUS.md §6.2
**Problem:** `pbft_check_timeouts()` references `entry->state_entered_ms` but the struct in §3.1 doesn't have that field.
**Fix:** add `uint64_t state_entered_ms;` to `pbft_log_entry_t` and update size.

### 🟠 B7. New-View size forces UDP even in ESP-NOW-only deployment
**File:** PROTOCOL.md §8.2
**Problem:** "esp-pbft uses Wi-Fi UDP for New-View even when ESP-NOW is the default transport" — implies UDP must be initialized at boot (≈ +30 KB RAM) even for ESP-NOW-only deployments. This contradicts the "low memory" pitch.
**Fix:** alternatives — (a) accept the +30 KB cost (still fits within 400 KB; budget becomes 70 KB total) (superseded R2: ~72 KB esp-pbft / ~117 KB grand), (b) design a fragmentation scheme for New-View (2-3 ESP-NOW frames), (c) reduce VC proof count by using a single combined VC-receipt per replica. Recommend (a) for v1 with a documented cost.

### 🟠 B8. No retransmit / resend logic for lost Prepare/Commit
**File:** CONSENSUS.md §6
**Problem:** ESP-NOW has ~5-10% loss in real environments; UDP/IGMP can lose more. PBFT paper assumes reliable broadcast (PBFT §5.5 acknowledges this is "not always true"). Lost Prepares cause view-change, which is correct but expensive.
**Fix:** add a simple retransmit strategy: if a Prepare doesn't reach quorum within `prepare_timeout_ms / 2`, rebroadcast. Same for Commit. Bounded by a small counter (≤ 3 retransmits) to avoid amplifying a Byzantine replica. Add to CONSENSUS.md §6.

### 🟠 B9. Adaptive timeout is described but not implemented
**File:** CONSENSUS.md §6.3
**Problem:** "Adaptive per round … start 1000 ms, +25% on timeout, -10% on commit." This requires per-entry feedback, but the design only has a single `prepare_timeout_ms` for the whole view.
**Fix:** either (a) implement as a single shared per-view value (simpler, still useful), or (b) mark as v2 feature. v1: simple per-view value, no adaptation. Document.

### 🟠 B10. No client-request dedup → vulnerable to duplicate execution by Byzantine primary
**File:** CONSENSUS.md §15 C2 (open)
**Problem:** a Byzantine primary can replay an old TX with a fresh sequence. Without dedup, the app may execute the same TX twice. Apps can be idempotent (digest-based), but esp-pbft should not rely on it.
**Fix:** add an LRU "executed digests" cache to `pbft_consensus.c` (e.g., last 256 digests, ~8 KB). On Pre-Prepare, if digest is in the cache and the corresponding seq is below high-watermark, log alarm + reject. New-View carries digests so a fresh primary can pre-populate the cache.

### 🟠 B11. Y-5 destroys keys before peers re-handshake → message loss window
**File:** CRYPTO.md §7.2
**Problem:** `psa_destroy_key(my_priv_key_id); mbedtls_platform_zeroize(hmac_keys, sizeof(hmac_keys));` is immediate, but peers take ~50 ms to re-ECDH after receiving the new Hello. During this window, any message the node tries to send (e.g., a Prepare in flight) has no valid MAC key.
**Fix:** (a) pause consensus participation for the duration of the re-handshake (set a "re-keying" flag; new Prepare/Commit are rejected until flag clears), OR (b) keep old keys for an additional grace period (e.g., 5 seconds) to handle in-flight messages. Recommend (b) for simplicity.

---

## C. Documentation / clarity issues

### 🟡 C1. Endianness asserted but no wire format compliance test
**File:** PROTOCOL.md §4.2
**Fix:** add to TEST-PLAN.md a round-trip test that sends a packet and verifies byte order on the receiver.

### 🟡 C2. NVS layout undefined
**Files:** HANDOVER §4 (`pbft_storage.h`), PROTOCOL §9.3
**Fix:** define namespace + keys + sizes in DEPLOYMENT.md.

### 🟡 C3. `pbft_metrics_t` referenced but undefined
**File:** HANDOVER §5
**Fix:** define in API-REFERENCE.md.

### 🟡 C4. Error codes not enumerated
**File:** HANDOVER §5, CONSENSUS §5.2 (`PBFT_ERR_NOT_PRIMARY`)
**Fix:** catalog in API-REFERENCE.md §3.

### 🟡 C5. `pbft_config_t` referenced but undefined
**File:** HANDOVER §5
**Fix:** define in API-REFERENCE.md §2.

### 🟡 C6. Throughput claim "30 TPS" is hand-waved
**File:** CONSENSUS.md §9
**Fix:** replace with measured formula: `TPS = 1 / (3 × RTT + processing)`. For ESP-NOW RTT ≈ 20 ms, processing ≈ 10 ms, TPS ≈ 1/70ms ≈ 14 TPS. For UDP RTT ≈ 40 ms, TPS ≈ 9. Document this honestly.

### 🟡 C7. TOFU weakness not explicitly listed as a deployment assumption
**File:** CRYPTO §10.3
**Fix:** add to DEPLOYMENT.md "physical security requirement" section.

### 🟡 C8. Light-sleep wake sources not specified
**File:** HANDOVER §3.8 mentions 0.5 mA light sleep but doesn't explain how the node wakes for incoming ESP-NOW messages.
**Fix:** POWER.md (to be written) addresses this — ESP-NOW wake-up uses promiscuous RX, can wake from light sleep via `WIFI_PKT` or `esp_now` callback.

### 🟡 C9. Anti-pattern A4 "pointer aliasing" is correctly banned but Pre-Prepare serialization needs explicit guidance
**File:** CODING_STANDARD.md
**Fix:** add worked example showing `memcpy` based serialization for packed structs.

### 🟡 C10. View-change state machine not in CONSENSUS
**File:** CONSENSUS.md
**Fix:** VIEW-CHANGE.md (to be written) provides full state machine for view-change phases.

### 🟡 C11. Checkpoint protocol not defined
**File:** CONSENSUS.md §8 only references
**Fix:** CHECKPOINT.md (to be written).

### 🟡 C12. No defined `pbft_packet_t` byte layout in code
**File:** PROTOCOL.md is prose only
**Fix:** in PROTOCOL.md, add `pbft_packet_t` C struct (packed) for each message type.

### 🟡 C13. Power budget "1 h active" is unjustified
**File:** HANDOVER §3.8
**Fix:** POWER.md provides a workload model (10 TPS sustained, etc.) and computes from that.

### 🟡 C14. No documentation of "what happens at boot when only 4 of 6 peers have arrived"
**File:** CRYPTO §4.1
**Fix:** define minimum quorum for boot completion (e.g., need 4-of-6 peers' Hellos to make any progress; otherwise stay in discovery loop).

---

## D. Nits

### 🟢 D1. `&(size_t){0}` compound-literal pointer in `psa_mac_compute` — works but ugly
**File:** CRYPTO.md §5.3 — use `size_t mac_len;` instead.

### 🟢 D2. "Magic number" 0xFF:FF:FF:FF:FF:FF for broadcast MAC — extract to constant
**File:** PROTOCOL.md §2.2.

### 🟢 D3. `pbft_log_get_or_create` not defined
**File:** CONSENSUS.md §4.2 — add to API-REFERENCE.

### 🟢 D4. `pbft_log_find` not defined
Same.

### 🟢 D5. `pbft_send_pre_prepare` etc. not declared anywhere
Same.

### 🟢 D6. Staggered boot "0-3s" magic number — extract to Kconfig

### 🟢 D7. Magic numbers in MAC truncation (32 bytes) — extract to `PBFT_MAC_BYTES`

### 🟢 D8. Y-5 stagger "node_id × 1 hour" — extract to Kconfig

---

## E. ESP32-C3 feasibility check

| Capability | Required | Available | OK? |
|------------|----------|-----------|-----|
| SRAM | 55-70 KB | 400 KB | ✅ 14-18% |
| Flash | ~200 KB code | 4 MB | ✅ 5% |
| TRNG | Yes (P-256 keygen) | Hardware TRNG | ✅ |
| SHA-256 | HMAC runtime | Hardware SHA | ✅ 3-5 µs/msg |
| ECC point-mul | ECDH | Hardware ECC | ✅ ~8 ms/keygen |
| Wi-Fi 2.4 GHz | ESP-NOW / UDP | Yes | ✅ |
| FreeRTOS task + queue | Event loop | Yes | ✅ |
| NVS | Persistent config | Yes | ✅ 1 KB needed |
| Light sleep + wake | Power saving | Yes (WIFI_PKT) | ✅ |
| ECDSA sign | NOT NEEDED | N/A | ✅ |

**Conclusion:** feasible. Memory budget requires care but is achievable.

---

## F. Required new documents (per this audit)

| File | Purpose | Priority |
|------|---------|----------|
| VIEW-CHANGE.md | Fix A1, A2; full protocol | 🔴 P0 |
| CHECKPOINT.md | Fix A6; define watermark advance | 🔴 P0 |
| API-REFERENCE.md | Fix C3, C4, C5; define all public API | 🔴 P0 |
| MEMORY.md | Fix A5; per-module layout, static_assert catalog | 🔴 P0 |
| POWER.md | Fix C8, C13; light-sleep state machine | 🟠 P1 |
| FAILURE-MODES.md | Fix B1; full Byzantine scenario catalogue | 🟠 P1 |
| TEST-PLAN.md | Define test strategy + heap-peak measurement | 🟠 P1 |
| DEPLOYMENT.md | Fix C2, C7; NVS schema, provisioning | 🟠 P1 |
| ROADMAP.md | Sequenced delivery plan | 🟡 P2 |
| INDEX.md | Master index | 🟡 P2 |

---

## G. Summary

- **25 design issues identified**, 7 protocol-breaking (must fix in the new docs).
- **All existing docs have genuine value** — they correctly identify Pattern Y, n=7/f=2/quorum=5, transport choice, and the static-memory discipline. The design is on a sound footing.
- **Implementation can begin** after VIEW-CHANGE.md, CHECKPOINT.md, API-REFERENCE.md, and MEMORY.md are written and the protocol-breaking issues (A1-A7) are addressed.
- The audit itself should be reviewed by the project owner before any new doc is merged.

**End of DESIGN-AUDIT.md (v0.1 — 2026-06-27)**