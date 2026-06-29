# esp-pbft — Implementation Roadmap

> **Status:** 🟡 Draft (v0.1)
> **Scope:** phased implementation plan with milestones, exit criteria, dependencies between phases, and staffing estimate.
> **Out of scope:** design → see other docs in `docs/`; API stability → [API-REFERENCE.md](./API-REFERENCE.md).

This document is the **canonical reference for the implementation plan**. It supersedes the high-level roadmap in HANDOVER §11.

---

## 1. Phasing strategy

esp-pbft is built in **5 phases over ~14 weeks**, with each phase producing a working, testable artefact. Phase gates are enforced — a phase cannot start until its predecessor's exit criteria are met.

| Phase | Duration | Deliverable | Exit gate |
|-------|----------|-------------|-----------|
| **P1. Skeleton** | 2 weeks | compiles, single-node boots, no consensus | `idf.py build` succeeds; `pbft_init()` returns OK on devkit |
| **P2. Core consensus** | 3 weeks | 3-phase PBFT works on 7-node cluster, happy path only | Host-harness T2.H2 passes (1000 TXs committed, no forks) |
| **P3. Hardening** | 3 weeks | view-change, checkpoint, Y-5, retransmit | T4.C3 passes (14 TPS × 10 min, no view-changes); T4.C4 passes (reboot recovery) |
| **P4. Testing & docs** | 3 weeks | Byzantine tests pass, full docs published | T5.BF7 passes (safety under 6-of-7 Byzantine); all docs reviewed |
| **P5. Release** | 2 weeks | v1.0 tagged, examples working, deployment guide validated | Production cluster runs 24 h soak test |
| **P6. Post-v1** (optional) | TBD | API stabilises, RFC submitted | API frozen for 6 months |

Total: ~13 weeks to v1.0, plus optional P6.

---

## 2. Phase 1 — Skeleton (weeks 1-2)

### 2.1 Goal

Establish the build system, project structure, and minimal API surface. No PBFT logic yet.

### 2.2 Tasks

| # | Task | File(s) | Days |
|---|------|---------|------|
| 1.1 | Set up `idf_component.yml` with ESP-IDF v6.0.1 dependency | root | 0.5 |
| 1.2 | Create `Kconfig` with all options (default values) | `pbft/Kconfig` | 0.5 |
| 1.3 | Create `pbft_config.h` with constants + `_Static_assert` catalog | `include/pbft_config.h` | 1 |
| 1.4 | Stub `pbft_init`, `pbft_start`, `pbft_stop`, `pbft_deinit` (return `ESP_OK`) | `src/pbft.c` | 1 |
| 1.5 | Implement basic FreeRTOS task creation (no logic yet) | `src/pbft.c` | 1 |
| 1.6 | Wire up `idf.py build` for `esp32c3` target | root CMake | 0.5 |
| 1.7 | Configure example app to build + flash + monitor | `example/counter/` | 1 |
| 1.8 | Smoke test: app boots, prints "PBFT ready" | (test) | 0.5 |

### 2.3 Exit criteria

- [ ] `idf.py build` succeeds with zero warnings on `esp32c3` target
- [ ] `idf.py -p /dev/ttyUSB0 flash monitor` shows "PBFT ready" within 5 s
- [ ] `cppcheck` reports zero defects
- [ ] All `_Static_assert`s in `pbft_config.h` pass

### 2.4 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| ESP-IDF v6.0.1 not installed | low | blocker | install from GitHub; document in README |
| Mbed TLS 4.0 PSA API differs from docs | medium | medium | validate against local install (`~/.espressif/v6.0.1/`) |

---

## 3. Phase 2 — Core consensus (weeks 3-5)

### 3.1 Goal

Implement the 3-phase PBFT protocol end-to-end on a 7-node cluster, in the happy path only (no failures).

### 3.2 Tasks

| # | Task | Module | Days |
|---|------|--------|------|
| 2.1 | PSA crypto init + TRNG keygen | `pbft_crypto.c` | 2 |
| 2.2 | Hello broadcast + receive + TOFU | `pbft_crypto.c`, `pbft_network.c` | 2 |
| 2.3 | ECDH + HMAC key derivation | `pbft_crypto.c` | 1 |
| 2.4 | HMAC compute + verify | `pbft_crypto.c` | 1 |
| 2.5 | ESP-NOW transport (init, broadcast, RX callback) | `pbft_network_espnow.c` | 2 |
| 2.6 | Wi-Fi UDP transport (init, multicast, RX callback) | `pbft_network_wifi_udp.c` | 2 |
| 2.7 | PBFT log (create, find, update) | `pbft_log.c` | 1 |
| 2.8 | `pbft_submit` (primary path) | `pbft_consensus.c` | 1 |
| 2.9 | `pbft_handle_pre_prepare` (replica path) | `pbft_consensus.c` | 1 |
| 2.10 | `pbft_handle_prepare` (quorum check) | `pbft_consensus.c` | 1 |
| 2.11 | `pbft_handle_commit` (quorum + callback) | `pbft_consensus.c` | 1 |
| 2.12 | State machine: timer for Prepare/Commit timeout | `pbft_consensus.c` | 1 |
| 2.13 | FreeRTOS event loop + queue | `pbft.c` | 1 |
| 2.14 | Host harness T2.H1 (all 7 boot) | `tools/host_harness/` | 2 |
| 2.15 | Host harness T2.H2 (1000 TXs) | `tools/host_harness/` | 2 |
| 2.16 | Cluster smoke T4.C1, C2 | `tools/cluster_test/` | 2 |

### 3.3 Exit criteria

- [ ] All ~50 unit tests pass
- [ ] T2.H1 passes (7 nodes boot to PBFT_STATE_NORMAL within 10 s)
- [ ] T2.H2 passes (1000 sequential TXs all committed, no forks)
- [ ] T4.C1 passes (7-node cluster boots on real hardware)
- [ ] T4.C2 passes (submission committed at all 7 within 200 ms)
- [ ] Average latency ≤ 50 ms (ESP-NOW)
- [ ] No memory leaks in 1-hour host-harness run

### 3.4 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| ESP-NOW broadcast not received by all peers | high | blocker | add retransmit (B8); debug with sniffer |
| PSA key generation fails on first boot | low | blocker | check `psa_crypto_init` returned `PSA_SUCCESS`; add retry |
| `pbft_handle_commit` callback fires during crypto update (re-entrancy) | medium | medium | document thread-safety contract; use queue if needed |

---

## 4. Phase 3 — Hardening (weeks 6-8)

### 4.1 Goal

Add fault tolerance: view-change, checkpoint/GC, Y-5 re-gen, retransmit, dedup cache.

### 4.2 Tasks

| # | Task | Module | Days |
|---|------|--------|------|
| 3.1 | View-Change message build/send | `pbft_viewchange.c` | 2 |
| 3.2 | View-Change receive + V-set accumulation | `pbft_viewchange.c` | 2 |
| 3.3 | New-View send (with V-set, O-set) | `pbft_viewchange.c` | 2 |
| 3.4 | New-View receive + verify + apply O-set | `pbft_viewchange.c` | 2 |
| 3.5 | O-set deterministic algorithm | `pbft_viewchange.c` | 2 |
| 3.6 | Checkpoint message build/send/receive | `pbft_checkpoint.c` | 1 |
| 3.7 | Watermark advance (low/high) | `pbft_checkpoint.c` | 1 |
| 3.8 | Log GC | `pbft_checkpoint.c` | 1 |
| 3.9 | State transfer (STATE-REQUEST/RESPONSE) | `pbft_checkpoint.c` | 2 |
| 3.10 | Y-5 re-gen timer + soft re-handshake | `pbft_crypto.c` | 1 |
| 3.11 | Y-5 grace period (audit B11) | `pbft_crypto.c` | 0.5 |
| 3.12 | Per-message retransmit (audit B8) | `pbft_consensus.c` | 1 |
| 3.13 | Executed-digest dedup cache (audit B10) | `pbft_consensus.c` | 1 |
| 3.14 | NVS save/restore (view, seq, watermarks) | `pbft_storage.c` | 1 |
| 3.15 | Metrics counters | `pbft.c` | 0.5 |
| 3.16 | Host harness T2.H4, H5, H8 | `tools/host_harness/` | 2 |
| 3.17 | Cluster test T4.C3 (10 min sustained), C4 (power-cycle) | `tools/cluster_test/` | 2 |

### 4.3 Exit criteria

- [ ] T2.H4 passes (primary down → view-change within 3 s)
- [ ] T2.H5 passes (replica crash + reconnect + catch-up)
- [ ] T2.H8 passes (24 h simulated time, all Y-5 fire correctly)
- [ ] T4.C3 passes (14 TPS × 10 min, ≤ 2 view-changes)
- [ ] T4.C4 passes (power-cycle node, recovers within 10 s)
- [ ] Heap peak < 8 KB transient (per POWER.md §4.4)
- [ ] No memory leaks in 6-hour host-harness run

### 4.4 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| View-change causes safety violation under specific timing | medium | catastrophic | extensive T5 Byzantine tests; replay-and-verify tool |
| Checkpoint GC frees entry that's still needed for O-set | low | catastrophic | gate GC on EXECUTED state only (CHECKPOINT.md §7.2) |
| Y-5 grace period misconfigured (messages still dropped) | medium | medium | extensive T2.H8 with simulated re-handshake failures |

---

## 5. Phase 4 — Testing & docs (weeks 9-11)

### 5.1 Goal

Validate Byzantine-fault tolerance end-to-end on real hardware, polish docs, ensure CI gate green.

### 5.2 Tasks

| # | Task | Module | Days |
|---|------|--------|------|
| 4.1 | Byzantine fault harness (BF1-BF7) | `tools/byzantium/` | 3 |
| 4.2 | Safety verification (commit-set equality across honest nodes) | `tools/byzantium/` | 2 |
| 4.3 | Liveness verification (cluster recovers after attack) | `tools/byzantium/` | 1 |
| 4.4 | T5.BF1 through T5.BF7 on real hardware | (test) | 3 |
| 4.5 | T7 perf benchmarks (latency, throughput) | `tools/` | 2 |
| 4.6 | T6 24-hour soak (subset, e.g. 1 hour) | `tools/` | 1 |
| 4.7 | Doc review (peer review of all docs) | (review) | 2 |
| 4.8 | Example app: counter (full implementation) | `example/counter/` | 1 |
| 4.9 | Example app: sensor telemetry | `example/sensor/` | 1 |
| 4.10 | Doxygen annotations on all public APIs | (code) | 1 |
| 4.11 | README.md updates + screenshots | root | 0.5 |
| 4.12 | GitHub Actions CI green | `.github/workflows/` | 1 |

### 5.3 Exit criteria

- [ ] All 7 Byzantine scenarios (BF1-BF7) pass safety + liveness checks
- [ ] T7 perf: P50 latency ≤ 50 ms, throughput ≥ 14 TPS (ESP-NOW)
- [ ] All 12 docs reviewed by 2 reviewers each
- [ ] CI gate green (unit + host integration + static analysis)
- [ ] Code coverage ≥ 80% (line), ≥ 75% (branch)
- [ ] No `cppcheck` or `clang-tidy` warnings

### 5.4 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Byzantine test reveals safety bug | medium | catastrophic | halt phase; return to P3 for fix; document the bug |
| Docs have stale references | medium | medium | doc review checklist; CI link check |
| CI flaky due to timing-sensitive tests | high | medium | add jitter to timeouts in tests; use statistical pass criteria |

---

## 6. Phase 5 — Release (weeks 12-13)

### 6.1 Goal

Tag v1.0, run final 24-hour soak test, publish release artifacts.

### 6.2 Tasks

| # | Task | Days |
|---|------|------|
| 5.1 | Final 24-hour soak test (T6) | 1 (run in background) |
| 5.2 | Performance baseline (T7 final) | 0.5 |
| 5.3 | Update CHANGELOG.md | 0.5 |
| 5.4 | GitHub release with artifacts (compiled firmware for esp32c3) | 0.5 |
| 5.5 | Component registry (`idf.py add-dependency "phukrit7171/esp-pbft"`) | 0.5 |
| 5.6 | Announce on Espressif forum / ESP32 subreddit | 0.5 |
| 5.7 | Tag v1.0.0 in git | 0.1 |

### 6.3 Exit criteria

- [ ] 24-hour soak passes (heap stable, all TXs committed, no view-change loops)
- [ ] CHANGELOG.md complete
- [ ] Component listed on `components.espressif.com`
- [ ] **External security review gate (AUDIT-FIX IR-8, 2026-06-29):**
  - [ ] 1 consultant-week external review of CRYPTO.md, FAILURE-MODES.md, DEPLOYMENT.md
  - [ ] Zero P0 (protocol-breaking) findings open at v1.0.0 tag
  - [ ] OR all P0 findings resolved before tag (each with a regression test)
  - [ ] P1 (security hardening) findings documented in `docs/v1.0.0-security.md`
- [ ] v1.0.0 tag pushed

---

## 7. Phase 6 — Post-v1 (optional)

### 7.1 Candidates

| Feature | Priority | Notes |
|---------|----------|-------|
| **Adaptive timeout** | medium | per-view value, +25% on timeout, -10% on commit |
| **TX batching** | high | 5-10× throughput improvement |
| **Persistent keys (PSA secure storage)** | medium | remove Y-5 grace period; faster recovery |
| **Multi-primary forwarding** (`pbft_submit_with_forward`) | medium | addresses audit A3 |
| **Adaptive view-change** | low | skip view-change if cluster obviously healthy |
| **BFT-SMaRt-style request ordering** | low | pipeline Prepare/Commit for higher throughput |
| **Multi-cluster federation** | low | requires cross-cluster PBFT (research) |
| **TLS for Hello (drop TOFU)** | high (security) | +50 ms handshake; drop audit B1 weakness |
| **Formal verification (TLA+ model)** | medium | proves safety properties |

### 7.2 Decision criteria

P6 features are taken up based on:
- User feedback (issues opened)
- Performance bottlenecks measured in production
- Security advisories

---

## 8. Staffing estimate

| Role | Effort | Where in timeline |
|------|--------|-------------------|
| Embedded C engineer (full-time) | 13 weeks | P1-P5 |
| Distributed systems engineer (part-time, 50%) | 13 weeks | P2-P5 (PBFT logic + tests) |
| DevOps / CI engineer (part-time, 25%) | 4 weeks | P4-P5 (CI gate + cluster test) |
| Technical writer (part-time, 25%) | 4 weeks | P4-P5 (docs) |
| Security reviewer (consultant) | 1 week | P5 (audit before release) |

**Total:** ~16 person-weeks over 13 calendar weeks.

---

## 9. Critical-path dependencies

```
P1 → P2 → P3 → P4 → P5 → (P6)
```

| Phase | Depends on |
|-------|------------|
| P2 | P1 (need build + project structure) |
| P3 | P2 (need working consensus to break it with failures) |
| P4 | P3 (need all features to test them) |
| P5 | P4 (need green CI + tests to release) |

P2 and P3 cannot overlap (need stable P2 to test P3 fault tolerance). P4 can overlap with P5.

---

## 10. Open questions (roadmap level)

| # | Question | Recommendation |
|---|----------|----------------|
| R1 | When to add secure boot V2 as default? | P5, before release. |
| R2 | When to add flash encryption as default? | P5, before release. |
| R3 | When to drop TOFU (audit B1)? | P6 (v1.1), requires RSA-2048 HW on C3 (DS peripheral supports it; ~50 ms handshake cost). |
| R4 | Should we target esp32c6 / esp32h2 in addition to esp32c3? | v1.1 (those chips are newer; verify PSA API compatibility first). |

---

## 11. References

- [HANDOVER.md](./HANDOVER.md) §11 — high-level roadmap (superseded)
- [TEST-PLAN.md](./TEST-PLAN.md) — test tiers used as P2/P3/P4 exit gates
- [DEPLOYMENT.md](./DEPLOYMENT.md) — what production deployment looks like (drives P5)
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — issues that must be fixed before each phase

---

**End of ROADMAP.md (v0.1 draft — phased implementation plan, ~13 weeks to v1.0)**