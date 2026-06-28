# esp-pbft — Failure Modes and Mitigations

> **Status:** 🟡 Draft (v0.1)
> **Scope:** Byzantine scenario catalogue; per-transport failure modes; detection and recovery actions; safety-vs-liveness trade-offs; physical-security assumptions.
> **Out of scope:** view-change details → [VIEW-CHANGE.md](./VIEW-CHANGE.md); checkpoint details → [CHECKPOINT.md](./CHECKPOINT.md); crypto details → [CRYPTO.md](./CRYPTO.md).

This document is the **canonical reference for failure handling in esp-pbft**. It addresses audit issue **B1** (TOFU weakness not explicit) and consolidates failure scenarios previously scattered across HANDOVER, CRYPTO, PROTOCOL, CONSENSUS, and ARCHITECTURE.

---

## 1. Threat model

### 1.1 Adversary capabilities

| Capability | Assumed? | Notes |
|------------|----------|-------|
| Eavesdrop on RF channel | ✅ assumed | passive monitoring of all packets |
| Inject arbitrary packets on RF channel | ✅ assumed | forgery attempts |
| Replay old packets | ✅ assumed | captured-packet replay |
| Partition the network | ✅ assumed | selective jamming / RF shielding |
| Delay packets arbitrarily | ✅ assumed | network congestion, adversarial delay |
| Control up to `f=2` cluster nodes | ✅ assumed (Byzantine model) | nodes may behave arbitrarily |
| Compromise TRNG | ❌ NOT assumed | ESP32-C3 TRNG is NIST SP 800-22 compliant |
| Extract RAM via physical access | 🟡 partially mitigated | Y-5 bounds the window to 24 h, JTAG disabled; not fully defended (see §5.6, DEPLOYMENT/TOFU §3.1) |
| MITM during boot before any Hello | ❌ NOT assumed | TOFU model — first Hello wins (see §3.1) |
| Compromise ESP-IDF Wi-Fi driver | ❌ NOT assumed | trust the SDK |
| Side-channel (power, EM) on HMAC | ❌ NOT assumed | PSA hardware mitigations |

### 1.2 What the system guarantees

Under the assumptions in §1.1, esp-pbft provides:

- **Safety:** all honest replicas commit the same TX at the same sequence (no forks, no double-execution)
- **Liveness:** valid TXs are eventually committed if the network is eventually synchronous and ≥ `2f+1 = 5` honest replicas can communicate
- **Authentication:** every PBFT message is bound to its sender (HMAC over `(view, seq, type, digest, payload)`)
- **Anti-replay:** old messages cannot be replayed (view + seq in MAC)
- **Bounded compromise window:** Y-5 re-gen limits exposure to 24 h

---

## 2. Byzantine scenarios (node-level)

Each scenario is rated for: detectability, automatic mitigation, residual risk.

### 2.1 Primary misbehavior

#### 2.1.1 Primary stops proposing

**Symptom:** app calls `pbft_submit()`, but no Pre-Prepare arrives at any replica.

**Detection:** replicas timeout on `prepare_timeout_ms = 1000 ms` (trigger T1 in VIEW-CHANGE §2).

**Mitigation:** view-change → new primary → resume. Worst-case latency: ~3 s.

**Residual risk:** none — PBFT paper proves recovery.

#### 2.1.2 Primary assigns conflicting sequences (equivocation)

**Symptom:** replica A receives Pre-Prepare `(v=0, seq=10, digest=X)`, replica B receives `(v=0, seq=10, digest=Y)`.

**Detection:** at each replica, Prepare digest must match its own Pre-Prepare digest. Mismatch → log alarm + skip Prepare. Neither digest reaches quorum of Prepares → view-change timeout → view-change.

**Mitigation:** view-change triggered by timeout. New primary picks one digest (the one with f+1 votes — but if neither has f+1 votes, O-set becomes no-op for that sequence).

**Residual risk:** one of X or Y may have been correctly prepared by an honest client → that TX is silently dropped. App must detect via dedup cache + retry. Audit B10 partially addresses.

#### 2.1.3 Primary sends Pre-Prepare with bad HMAC

**Symptom:** HMAC verify fails on all 6 replicas.

**Detection:** silent drop at transport layer (PROTOCOL §11).

**Mitigation:** timeout → view-change.

**Residual risk:** wasted time (~1 s before timeout). If primary is Byzantine, this is expected behaviour.

#### 2.1.4 Primary sends Pre-Prepare with sequence gap

**Symptom:** primary assigns sequences 1, 2, 4 (skips 3).

**Detection:** replicas reject (CONSENSUS §5.3 "gap-free assignment").

**Mitigation:** timeout → view-change. New primary assigns 3 then resumes.

**Residual risk:** honest primary may have skipped due to internal bug — but better safe than sorry.

### 2.2 Replica misbehavior

#### 2.2.1 Replica sends Prepare/Commit for TX it never received

**Symptom:** replica votes on a TX without first observing Pre-Prepare.

**Detection:** other replicas count Prepares only from senders whose Pre-Prepare matched. Byz replica's Prepare is recorded but counted only if its digest matches the agreed digest (which it might).

**Mitigation:** if the Byz replica's vote is for a digest that no honest replica agrees with, the vote contributes to a conflicting-quorum that doesn't reach 2f+1 — view-change.

**Residual risk:** with f=2, a Byz replica can match at most 1 honest replica's vote. Quorum of 5 still requires 3 honest votes — Byzantine quorum impossible.

#### 2.2.2 Replica equivocation (different Prepare to different peers)

**Symptom:** replica X sends Prepare(seq=10, digest=A) to replicas 1, 2, 3 and Prepare(seq=10, digest=B) to replicas 4, 5, 6.

**Detection:** each replica counts Prepares by digest. The "wrong" side sees only 1 (X) Prepare for its digest (assuming X only sent B there); the "right" side sees X + its own Prepares for A.

**Mitigation:** quorum is on a per-digest basis. Equivocation cannot push a wrong digest to quorum (would need 5 votes for B, but at most 2 Byzantine nodes can vote B — X plus one more). 3 honest votes for A is enough for A's quorum.

**Residual risk:** log alarm (audit B + Forensics).

#### 2.2.3 Replica sends stale Prepare (old view)

**Symptom:** replica X sends Prepare(view=0, seq=10) when current view is 5.

**Detection:** replicas reject (CONSENSUS.md — verify view matches current).

**Mitigation:** silent drop. Counter incremented (`pbft_metrics_t.mac_failures` may not increment — view check happens before MAC verify).

**Residual risk:** wasted airtime.

#### 2.2.4 Replica doesn't respond (silent crash)

**Symptom:** replica X stops sending Prepares, Commits, Checkpoints.

**Detection:** by the other replicas via timeouts.

**Mitigation:** if X is the primary, view-change. If X is a replica, its absence reduces the active quorum but doesn't stop consensus (need 5 of 6 non-primary votes; missing 1 is OK).

**Residual risk:** if 2 replicas crash simultaneously (one primary, one replica), view-change works (5 honest left). If 3 crash, liveness halts (only 4 left, < quorum 5).

#### 2.2.5 Replica sends duplicate Prepare (same digest, same view, same seq, same sender)

**Symptom:** two Prepares arrive from X for the same `(v, seq, digest)`.

**Detection:** `prepare_bitmask` is already set for X → second Prepare ignored.

**Mitigation:** built-in dedup.

**Residual risk:** none.

### 2.3 Combined Byzantine attacks

#### 2.3.1 Primary + 1 replica collude

**Scenario:** primary P sends Pre-Prepare(seq=10, digest=evil), replica R sends Prepare for evil digest. The other 4 honest replicas see different digests from R (since R is Byzantine, it might send evil to some and nothing to others).

**Detection:** quorum check — at most 2 votes for evil (P + R); need 5.

**Mitigation:** view-change. New primary re-proposes with the honest digest (if any honest replica prepared one).

**Residual risk:** if the 4 honest replicas never prepared either (because P's Pre-Prepare was rejected), the seq becomes a no-op in the O-set. The "evil" TX is dropped silently.

#### 2.3.2 Two replicas collude (equivocation)

**Scenario:** replicas R1 and R2 each equivocate between two subsets of the cluster.

**Detection:** quorum math — each subset sees at most 2 equivocating votes for any digest.

**Mitigation:** no effect on safety. View-change may be triggered by honest replicas if their prepared certificates are inconsistent.

**Residual risk:** minor throughput loss.

#### 2.3.3 Two replicas + primary all malicious

**Scenario:** 3 of 7 nodes Byzantine (exceeds f=2 limit → safety not guaranteed).

**Detection:** none possible in PBFT model.

**Mitigation:** none — safety violation possible. The design's whole point is to tolerate up to f=2.

**Residual risk:** **catastrophic** — system is outside its threat model.

---

## 3. Network-level failures

### 3.1 ESP-NOW-specific

#### 3.1.1 Broadcast storm at boot

**Symptom:** 7 nodes boot simultaneously → 7 simultaneous Hellos → packet collisions.

**Detection:** some Hellos lost.

**Mitigation:** boot staggering (POWER.md §5.2) — each node delays 0-3 s randomly.

**Residual risk:** in rare cases, all 7 nodes randomize to similar delays. Mitigation: 5 s discovery timeout → retry.

#### 3.1.2 ESP-NOW peer table overflow

**Symptom:** adding 8th peer fails (ESP-NOW supports 20 encrypted, 6 unencrypted).

**Detection:** `esp_now_add_peer` returns error.

**Mitigation:** use encrypted mode (CCMP is built-in for encrypted peers; ESP-NOW supports up to 20 encrypted peers — sufficient for n=7).

**Residual risk:** none at n=7.

#### 3.1.3 Channel interference / 2.4 GHz noise

**Symptom:** high packet loss (> 30%).

**Detection:** repeated retransmits, view-change timeouts.

**Mitigation:** automatic retransmit (PBFT-LEVEL) — see §3.4. Physical layer (802.11) handles per-frame retransmit; ESP-NOW has 3 internal retries.

**Residual risk:** under heavy interference, may not be recoverable. esp-pbft documents assumption: "cluster deployed in low-interference environment."

#### 3.1.4 Range exceeded

**Symptom:** nodes at the edge of ESP-NOW range (~50 m nominal, 200 m max line-of-sight).

**Detection:** same as §3.1.3.

**Mitigation:** physical repositioning. Software has no answer.

### 3.2 Wi-Fi UDP-specific

#### 3.2.1 AP crashes

**Symptom:** the AP (typically node 0) dies → STAs (other nodes) lose association.

**Detection:** STAs report disconnect via Wi-Fi event.

**Mitigation:** if AP node is also PBFT primary, view-change → new primary starts its own AP (if it's the new AP role) or STAs find another AP. **Note:** esp-pbft's UDP design assumes ONE AP at a time; if the AP node dies and a STA takes over, all other STAs must re-associate. This may take 1-2 s.

**Residual risk:** liveness pause during re-association.

#### 3.2.2 IGMP group leave

**Symptom:** a node leaves the multicast group → it misses subsequent PBFT messages.

**Detection:** missing messages → timeouts → view-change.

**Mitigation:** node must re-join IGMP group on Wi-Fi reconnect.

**Residual risk:** none if re-join is automatic (ESP-IDF handles).

#### 3.2.3 DHCP lease expiry

**Symptom:** a node's IP lease expires → cannot send unicast.

**Detection:** `sendto` returns error or times out.

**Mitigation:** lease renewal; `esp_netif` handles automatically if DHCP client enabled.

**Residual risk:** none.

### 3.3 Common (both transports)

#### 3.3.1 Network partition — minority side

**Symptom:** 3 honest + 2 Byzantine on one side; 4 honest on the other.

**Detection:** minority side cannot reach quorum.

**Mitigation:** VIEW-CHANGE.md §9.3 — declare partition mode after MAX_VIEW_GAP escalations.

**Residual risk:** liveness halts on minority side until partition heals.

#### 3.3.2 Network partition — majority side

**Symptom:** 5 honest + 2 Byzantine on one side; 2 honest on the other.

**Detection:** majority proceeds; minority stuck.

**Mitigation:** majority continues; minority eventually joins back via soft re-handshake.

**Residual risk:** none on majority; minority has liveness pause.

#### 3.3.3 Asymmetric partition

**Symptom:** A can send to B but B cannot send to A.

**Detection:** B sees Pre-Prepares from A but A never sees B's Prepares.

**Mitigation:** A's view-change triggers if Prepare quorum not reached. View-change messages from B may not reach A → A sends VIEW-CHANGE(v+1) which B doesn't receive. If B sees 2f+1 VIEW-CHANGEs from other replicas, B progresses; A eventually escalates to v+k until it sees a New-View.

**Residual risk:** slow recovery (multiple view-change rounds).

#### 3.3.4 Single-node network outage

**Symptom:** node X disconnects for 30 s.

**Detection:** other nodes' timeouts.

**Mitigation:** cluster proceeds (X is "absent" — quorum of 5 still reachable). When X reconnects:
- TRNG re-key (soft re-handshake)
- NVS recovery (view/seq from last clean shutdown)
- Catch up via STATE-RESPONSE

**Residual risk:** X misses TXs during outage; app state on X is stale until catch-up.

#### 3.3.5 Replay attack (captured Prepare rebroadcast)

**Symptom:** attacker captures Prepare(v=0, seq=10, digest=X) and rebroadcasts later.

**Detection:** `view+seq` in MAC; old Prepare rejected (current view is v=5, not v=0).

**Mitigation:** built-in (CRYPTO §5.2).

**Residual risk:** none.

#### 3.3.6 MAC forgery

**Symptom:** attacker without HMAC key sends Prepare claiming to be from node 5.

**Detection:** HMAC verify fails.

**Mitigation:** silent drop.

**Residual risk:** none (key required).

### 3.4 Retransmit policy

PBFT-level retransmit (audit B8). One budget everywhere: `PBFT_MAX_RETRIES = 3`, tracked by
the per-entry `retry_count` counter (NOT a one-shot flag). A phase rebroadcasts when it has
not reached quorum within `timeout_ms / 2` **and** `retry_count < PBFT_MAX_RETRIES`; each
rebroadcast does `retry_count++`. Quorum is `__builtin_popcount(entry->prepare_bitmask)` /
`commit_bitmask` ≥ `2f+1` (there are no `prepare_count` / `commit_count` fields).

| Message | Retransmit trigger | Max retries |
|---------|--------------------|-------------|
| Prepare | local entry in PRE_PREPARED for `> prepare_timeout_ms / 2`, quorum not reached, and `retry_count < PBFT_MAX_RETRIES` | `PBFT_MAX_RETRIES = 3` |
| Commit | local entry in PREPARED for `> commit_timeout_ms / 2`, quorum not reached, and `retry_count < PBFT_MAX_RETRIES` | `PBFT_MAX_RETRIES = 3` |
| Checkpoint | local has not received quorum for `> checkpoint_timeout_ms / 2` and `retry_count < PBFT_MAX_RETRIES` | `PBFT_MAX_RETRIES = 3` |
| View-Change | no New-View received within `viewchange_timeout_ms / 2` and `retry_count < PBFT_MAX_RETRIES` | `PBFT_MAX_RETRIES = 3` |
| New-View | (new primary) no replica has sent any Prepare for `> viewchange_timeout_ms / 2` and `retry_count < PBFT_MAX_RETRIES` | `PBFT_MAX_RETRIES = 3` |

After `retry_count` reaches `PBFT_MAX_RETRIES` without quorum, the entry's timeout escalates
to a view-change (Prepare/Commit) rather than rebroadcasting indefinitely.

Jitter: ± 10% random to avoid synchronised retransmit storms.

---

## 4. Node-level failures

### 4.1 Clean reboot

**Trigger:** firmware update, `esp_restart()`.

**Sequence:**
1. `pbft_stop()` → NVS flush (view, seq, watermarks)
2. `esp_restart()` → reboot
3. New TRNG keypair (Pattern Y)
4. Broadcast Hello (peers see new pubkey → soft re-handshake)
5. Resume from saved view (or fall back to view-change if cluster has advanced)

**Duration:** ~3.5 s total (POWER.md §5.1).

### 4.2 Crash (watchdog reset)

**Trigger:** task watchdog, hardware watchdog.

**Sequence:** same as clean reboot but NVS not flushed. Cluster sees node "disappear" for ~3.5 s, then re-handshake.

**Recovery:** automatic via soft re-handshake. Node catches up via STATE-RESPONSE.

### 4.3 Power loss (brownout)

**Trigger:** battery drained below 3.0 V.

**Sequence:** same as crash. On power restore, boots with last clean NVS state (or initial state if never shut down cleanly).

**Recovery:** automatic.

### 4.4 Flash corruption

**Trigger:** bad block in OTA partition.

**Detection:** bootloader fails to verify → rolls back to previous firmware.

**Mitigation:** factory + OTA_0 + OTA_1 partitions; rollback automatic.

**Residual risk:** if factory partition corrupted → node bricks (recovery requires physical reflash).

### 4.5 NVS corruption

**Trigger:** power loss during NVS write.

**Detection:** NVS read returns garbage → esp-pbft refuses to start.

**Mitigation:** NVS uses wear-levelling + CRC; corruption rare. If it happens, factory reset + re-provision.

**Residual risk:** low (NVS designed for power-loss safety).

---

## 5. Crypto-specific failures

### 5.1 TOFU weakness (audit B1)

**Threat:** MITM at boot substitutes a node's pubkey before the legitimate node's Hello arrives.

**Defence in current design:** none — TOFU trusts the first Hello per channel.

**Mitigation options:**

| Option | Cost | Recommendation |
|--------|------|----------------|
| Document physical-security requirement | 0 | ✅ v1 — document in DEPLOYMENT.md |
| Manufacturer-injected RSA-2048 signature on Hello | +50 ms handshake | ❌ v1 — adds latency |
| Operator-attended bootstrap | operational | ❌ v1 — manual scaling issue |
| Compare pubkey against out-of-band channel (QR code, NFC) | +10 s provisioning | ❌ v1 — slow |

**Decision (v1):** document as a deployment assumption. Operator must verify that all nodes are on the same RF channel during initial bootstrap.

### 5.2 Y-5 re-handshake race (audit B11)

**Threat:** node destroys its old keys before peers re-ECDH → in-flight messages fail.

**Current pseudocode:** immediate destroy.

**Mitigation (G9, option b — keep old keys; no consensus pause):**
- On re-gen: derive the new keys, then **KEEP** the old `hmac_keys[]` for the grace window
  (`PBFT_Y5_GRACE_MS = 5000`, 5 s). Do **not** `psa_destroy_key()` the old key immediately.
- During the grace window, accept a MAC verified under **either** the old or the new key.
- Send messages with the new key only.
- Schedule destruction of the old key after `PBFT_Y5_GRACE_MS`; the grace timer (not the
  re-gen step) is what calls `psa_destroy_key()`.
- Consensus is **NOT** paused — messages continue to be produced and verified throughout the
  grace window.

**Decision:** add 5 s keep-old-keys grace period in v1. See CRYPTO §7.2/§7.3.

### 5.3 PSA key attribute misconfiguration (audit B4)

**Threat:** key accidentally given `PSA_KEY_USAGE_SIGN_HASH` → stolen key can sign arbitrary messages.

**Current design:** usage is `DERIVE` only. Correct.

**Mitigation:** `// NOTE: NEVER add USAGE_SIGN_HASH here` comment in source. Add static_assert that the key attributes are DERIVE-only.

### 5.4 ECDH with untrusted peer pubkey

**Threat:** malicious peer sends a non-P-256 pubkey (e.g., point at infinity, small-subgroup point).

**Detection:** PSA validates during `psa_import_key` — returns error for invalid curve points.

**Mitigation:** check return value of `psa_import_key`. Log alarm + drop peer.

### 5.5 Replay of old Y-5 Hello

**Threat:** attacker captures old Hello (with old pubkey) and replays.

**Detection:** HMAC over Hello uses current `hmac_keys[sender]`. If peer's HMAC key has rotated since capture, MAC verify fails. If not yet rotated, MAC succeeds but pubkey differs from cached → CRYPTO §6.2 detects.

**Mitigation:** built-in.

### 5.6 Partial key compromise

**Threat:** attacker reads RAM via JTAG or other side-channel → obtains current ECDSA private key.

**Detection:** none possible (key is in plaintext in RAM).

**Mitigation:**
- Disable JTAG in production (`CONFIG_ESP_SECURE_JTAG_DISABLE=y`)
- Y-5 limits compromise window to 24 h
- Reboot generates new keypair

**Residual risk:** if compromise is persistent (backdoor installed), 24 h window is the limit.

---

## 6. Concurrency / timing failures

### 6.1 Two nodes think they are primary (split brain at view-change boundary)

**Threat:** during view-change from v to v+1, both old primary and new primary broadcast Pre-Prepares.

**Detection:** replicas are in `PBFT_VIEW_PHASE_WAIT_NEW` → reject any Pre-Prepare for old view.

**Mitigation:** built-in via view state machine.

### 6.2 Clock skew between nodes

**Threat:** ESP32-C3 has no RTC; uses ESP timer (derived from APB clock). Skew between nodes can cause premature timeout.

**Detection:** not directly detected.

**Mitigation:** keep timeouts generous (1000 ms prepare, 2000 ms viewchange). Skew typically < 10 ms — well within margins.

### 6.3 FreeRTOS task starvation

**Threat:** higher-priority task blocks PBFT task from running.

**Detection:** PBFT timeouts fire (esp_timer runs in interrupt context).

**Mitigation:** PBFT task priority = 5 (configurable). App tasks should be ≤ 4.

### 6.4 ISR during consensus

**Threat:** ISR fires mid-handler, re-enters same handler (re-entrancy).

**Detection:** esp-pbft is single-threaded by design; ISRs cannot call esp-pbft APIs.

**Mitigation:** use `pbft_submit_from_isr` (which queues, doesn't call directly). All esp-pbft APIs non-ISR-safe.

---

## 7. Safety vs liveness trade-offs

The PBFT paper and our design make several explicit trade-offs:

| Choice | Favours | Cost |
|--------|---------|------|
| Single-submitter (audit A3) | Simplicity | Primary SPOF for submissions |
| 2f+1 quorum (not n) | Liveness during 1-node outage | Higher per-message latency |
| Y-5 every 24 h | Bounded compromise window | Re-handshake complexity |
| Light-sleep wake on any 802.11 frame | Fast wake-up | False wake-ups from non-PBFT traffic (~rare) |
| No retransmit in v1 | Simplicity | Sensitivity to packet loss |
| Inline payload in PBFT log | Static memory | Higher RAM per entry |
| TOFU | No factory key injection | Boot MITM vulnerability (documented) |
| Single-threaded | No locks | Lower throughput ceiling |

---

## 8. Failure detection summary

| Symptom | Detection mechanism | Latency to detect | Action |
|---------|---------------------|-------------------|--------|
| Primary down | per-entry timeout | 1000 ms | view-change |
| Replica down | missing Prepare/Commit | 1000 ms | (none — cluster proceeds) |
| Network partition | timeout cascade | 1000-3000 ms | partition mode |
| HMAC failure | verify at RX | < 1 ms | drop + log |
| Pubkey mismatch | TOFU check at RX | < 1 ms | alarm + drop |
| RAM low | heap monitor | 60 s | log alarm |
| NVS corrupt | read failure | < 100 ms | refuse to start |
| TRNG failure | psa_generate_key return | < 10 ms | refuse to start |

---

## 9. Recovery playbook (operator-facing)

| Symptom | Operator action |
|---------|----------------|
| One node unresponsive for > 1 min | power-cycle the node; soft re-handshake will reconnect it |
| Cluster stuck (no commits for > 10 s) | check RF environment; force view-change via NVS (set `view` to current + 1 on one node) |
| Node reports "No peers" | check Wi-Fi/ESP-NOW init; check peer MAC table |
| Battery drains in < 1 day | check TX rate (may be runaway app); reduce via app throttling |
| Cluster forks (different views on different nodes) | recovery requires manual view synchronisation (set all nodes' NVS `view` to same value) — extremely rare, indicates bug |

---

## 10. Open questions (failure-mode level)

| # | Question | Recommendation |
|---|----------|----------------|
| F1 | Should we add a "view-change alarm" that operator can subscribe to? | v2 — out of scope for v1. |
| F2 | Should esp-pbft participate in leader election with Raft/PoS as fallback? | No — adds complexity; defeats PBFT's purpose. |
| F3 | Is Y-5 sufficient for physical-attack scenarios? | Depends on threat model. For high-security: consider RSA-signed Hello (v2). |
| F4 | Should we implement Byzantine-fault-detection (identify faulty node byz)? | Optional v2 — fingerprinting via behavior analysis. |

---

## 11. References

- [HANDOVER.md](./HANDOVER.md) — high-level design
- [CRYPTO.md](./CRYPTO.md) §10 — threat model (partial)
- [VIEW-CHANGE.md](./VIEW-CHANGE.md) — primary-fault recovery
- [CHECKPOINT.md](./CHECKPOINT.md) — state proof + catch-up
- [PROTOCOL.md](./PROTOCOL.md) — wire format
- [POWER.md](./POWER.md) §9 — brownout recovery
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — B1 (TOFU), B3 (zero MAC), B8 (retransmit), B11 (Y-5 grace)
- Castro & Liskov, "Practical Byzantine Fault Tolerance" (OSDI 1999), §5 — safety argument
- Lamport, "Byzantine Generals Problem" (1982) — original problem statement

---

**End of FAILURE-MODES.md (v0.1 draft — addresses audit issue B1 and consolidates failure analysis)**