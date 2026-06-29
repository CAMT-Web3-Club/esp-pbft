# esp-pbft — Test Plan

> **Status:** 🟡 Draft (v0.1)
> **Scope:** unit / integration / Byzantine / performance / 24-hour soak tests; fault-injection harness for the 7-node cluster; CI gate criteria; heap-peak and latency budgets.
> **Out of scope:** benchmarks for crypto alone → [CRYPTO.md](./CRYPTO.md) §8; power benchmarking → [POWER.md](./POWER.md).

This document is the **canonical reference for testing esp-pbft**. It addresses the missing TEST-PLAN.md referenced in HANDOVER §6 and covers the heap-peak measurement requirement called out in [MEMORY.md §9](./MEMORY.md).

---

## 1. Test tiers

| Tier | Target | Environment | Speed | Determinism |
|------|--------|-------------|-------|-------------|
| **T1. Unit** | A single source file's functions | Host (x86 Linux) + ESP-IDF Mocks | Seconds | Deterministic (seeded RNG) |
| **T2. Host integration** | Full PBFT state machine on host | x86 Linux, simulated peers via UDP loopback | Seconds to minutes | Deterministic with seeded RNG |
| **T3. Single-node smoke** | `pbft_init` → `pbft_start` → `pbft_submit` → callback fires | ESP32-C3 devkit, no peers | Seconds | Deterministic (single node) |
| **T4. Cluster integration** | 7-node cluster, end-to-end commits | 7 × ESP32-C3 devkits | Minutes | Stochastic (RF noise) |
| **T5. Byzantine injection** | f=1 and f=2 node fault injection | 7 × ESP32-C3 + fault harness | Hours | Replayable (logged packets) |
| **T6. Soak (24 h)** | Memory leaks, Y-5 re-gen, heap pressure | 7 × ESP32-C3 | Days | Statistical |
| **T7. Performance** | Throughput, latency, jitter | 7 × ESP32-C3 + sniffer | Hours | Statistical |

---

## 2. T1 — Unit tests

Framework: **Unity** (the test framework bundled with ESP-IDF's `idf.py test`).

Location: `test/` directory in the esp-pbft repo.

### 2.1 Coverage matrix

| Module | Test file | Test count target | Key cases |
|--------|-----------|-------------------|-----------|
| `pbft_config` | `test_config.c` | 10 | invalid node_id, missing fields, payload_max overflow |
| `pbft_crypto` | `test_crypto.c` | 25 | HMAC compute/verify roundtrip, ECDH determinism, peer import error |
| `pbft_membership` | `test_membership.c` | 8 | 7-node table, primary rotation |
| `pbft_consensus` | `test_consensus.c` | 50 | state machine transitions, quorum math, log full, gap detection |
| `pbft_viewchange` | `test_viewchange.c` | 30 | triggers, V-set/O-set, New-View verification, partition mode |
| `pbft_checkpoint` | `test_checkpoint.c` | 20 | watermark advance, GC, digest mismatch, state transfer |
| `pbft_network` | `test_network.c` | 18 | packet size limits, sender_id extraction, wire byte-order round-trip, wire-version mismatch reject |
| `pbft_log` | `test_log.c` | 10 | FIFO eviction, ring buffer wraparound |
| **Total** | — | **~173 tests** | — |

### 2.2 Example: HMAC round-trip

```c
TEST_CASE("HMAC compute/verify round-trip with same key", "[crypto]") {
    uint8_t key[32];
    for (int i = 0; i < 32; i++) key[i] = i;

    const uint8_t payload[] = "hello PBFT";
    uint8_t mac[32];
    size_t mac_len;

    TEST_ASSERT_EQUAL(PSA_SUCCESS,
        psa_mac_compute(
            hmac_key_id, PSA_ALG_HMAC(PSA_ALG_SHA_256),
            payload, sizeof(payload) - 1,
            mac, sizeof(mac), &mac_len));
    TEST_ASSERT_EQUAL(32, mac_len);

    TEST_ASSERT_EQUAL(PSA_SUCCESS,
        psa_mac_verify(
            hmac_key_id, PSA_ALG_HMAC(PSA_ALG_SHA_256),
            payload, sizeof(payload) - 1,
            mac, mac_len));
}
```

### 2.3 Example: quorum math

```c
TEST_CASE("quorum math: 5 of 7 = quorum", "[consensus]") {
    uint8_t bitmask = 0;
    TEST_ASSERT_FALSE(pbft_has_quorum(bitmask));
    bitmask |= (1u << 0);
    TEST_ASSERT_FALSE(pbft_has_quorum(bitmask));
    // ... add 4 more bits ...
    bitmask |= (1u << 4);
    TEST_ASSERT_TRUE(pbft_has_quorum(bitmask));
}
```

### 2.4 Example: wire byte-order (endianness) round-trip

Verifies the wire format is little-endian on the bus (DESIGN-AUDIT C1). Serialize a
Pre-Prepare, then inspect the raw bytes of every multi-byte field at its known offset and
assert little-endian layout, independent of host endianness. Also confirms a clean
deserialize round-trip.

```c
TEST_CASE("wire byte-order: multi-byte fields are little-endian", "[network]") {
    pbft_pre_prepare_t pp = {
        .hdr = { .msg_type = PBFT_MSG_PRE_PREPARE, .sender_id = 0,
                 .proto_version = PBFT_WIRE_VERSION, .reserved = 0 },
        .view = 0x11223344u,
        .sequence = 0x0102030405060708ull,
    };
    for (int i = 0; i < 32; i++) pp.digest[i] = (uint8_t)i;

    uint8_t buf[PBFT_MAX_PACKET];
    size_t n = pbft_serialize_pre_prepare(&pp, buf, sizeof(buf));
    TEST_ASSERT_GREATER_THAN(0, n);

    // view (u32 LE) at its offset: low byte first
    const uint8_t *v = buf + PBFT_OFF_VIEW;
    TEST_ASSERT_EQUAL_HEX8(0x44, v[0]);
    TEST_ASSERT_EQUAL_HEX8(0x33, v[1]);
    TEST_ASSERT_EQUAL_HEX8(0x22, v[2]);
    TEST_ASSERT_EQUAL_HEX8(0x11, v[3]);

    // sequence (u64 LE) at its offset: low byte first
    const uint8_t *s = buf + PBFT_OFF_SEQUENCE;
    TEST_ASSERT_EQUAL_HEX8(0x08, s[0]);
    TEST_ASSERT_EQUAL_HEX8(0x07, s[1]);
    TEST_ASSERT_EQUAL_HEX8(0x06, s[2]);
    TEST_ASSERT_EQUAL_HEX8(0x05, s[3]);
    TEST_ASSERT_EQUAL_HEX8(0x04, s[4]);
    TEST_ASSERT_EQUAL_HEX8(0x03, s[5]);
    TEST_ASSERT_EQUAL_HEX8(0x02, s[6]);
    TEST_ASSERT_EQUAL_HEX8(0x01, s[7]);

    // round-trip back to host order
    pbft_pre_prepare_t out;
    TEST_ASSERT_EQUAL(ESP_OK, pbft_deserialize_pre_prepare(buf, n, &out));
    TEST_ASSERT_EQUAL_UINT32(pp.view, out.view);
    TEST_ASSERT_EQUAL_UINT64(pp.sequence, out.sequence);
    TEST_ASSERT_EQUAL_MEMORY(pp.digest, out.digest, 32);
}
```

### 2.5 Example: wire-version mismatch rejected

Freezes the v1 wire format (G3 / PROTOCOL §4.1): any packet whose `proto_version` differs
from `PBFT_WIRE_VERSION` is rejected and counted as a dropped/invalid packet.

```c
TEST_CASE("wire-version mismatch is rejected", "[network]") {
    uint8_t buf[PBFT_MAX_PACKET];
    pbft_pre_prepare_t pp = make_valid_pre_prepare();   // proto_version = PBFT_WIRE_VERSION
    size_t n = pbft_serialize_pre_prepare(&pp, buf, sizeof(buf));

    // corrupt the proto_version byte in the common header
    buf[PBFT_OFF_PROTO_VERSION] = PBFT_WIRE_VERSION + 1;

    uint32_t dropped_before = s_metrics.invalid_packets;
    pbft_pre_prepare_t out;
    TEST_ASSERT_NOT_EQUAL(ESP_OK, pbft_deserialize_pre_prepare(buf, n, &out));
    TEST_ASSERT_EQUAL_UINT32(dropped_before + 1, s_metrics.invalid_packets);
}
```

### 2.6 CI gate

```
./tools/run_unit_tests.sh
# Expected: all tests pass; coverage report >= 80%
```

Coverage target: **80% line coverage** for all public modules.

---

## 3. T2 — Host integration tests

A full PBFT instance runs on x86 Linux using a mocked ESP-IDF layer. Peers communicate via UDP loopback. Tests cover end-to-end scenarios in seconds.

### 3.1 Host harness

`tools/host_harness/` provides:
- `mock_esp_now.{c,h}` — emulates `esp_now_*` over UDP loopback
- `mock_psa.{c,h}` — wraps PSA Crypto with deterministic key derivation (for reproducibility)
- `mock_timer.{c,h}` — virtual time, advanced manually
- `mock_nvs.{c,h}` — NVS backed by a file

### 3.2 Scenario tests

| # | Scenario | Expected outcome |
|---|----------|------------------|
| H1 | 7 nodes boot, all healthy | all 7 PBFT instances reach PBFT_STATE_NORMAL within 5 s |
| H2 | Submit 1000 TXs sequentially | all 1000 committed at all 7 replicas; no forks |
| H3 | Submit 1000 TXs concurrently | same as H2 (with random backoff) |
| H4 | Primary fails (no Pre-Prepare) | view-change within 3 s; cluster resumes |
| H5 | One replica crashes mid-protocol | cluster continues; crashed node catches up on reconnect |
| H6 | 2 of 7 nodes become Byzantine (equivocate) | safety holds; cluster may stall briefly but recovers via view-change |
| H7 | Network partition (3 | 4) | minority declares partition mode; majority proceeds |
| H8 | 24h simulated time (Y-5 fires 7 times) | all 7 nodes remain operational; no key material leaked |
| H9 | 100K TXs at max throughput | throughput stable; heap_min_free_bytes ≥ 16 KB throughout |
| H10 | Replay old Prepare | rejected (anti-replay via view+seq) |
| H11 | Submit duplicate TX (same payload) to primary | primary returns the **same** sequence both times (dedup cache hit, G6/B10); only one commit at every replica |
| H12 | Byzantine primary replays a committed digest in a new Pre-Prepare | every honest replica rejects it + logs alarm (digest cached ≤ high_watermark, G6/B10) |
| H13 | Boot with 4 of 6 peers present | quorum reached; the 4 proceed to PBFT_STATE_NORMAL (PBFT_BOOT_MIN_PEERS=4, C14) |
| H14 | Boot with 3 of 6 peers present | stays in discovery; no node reaches PBFT_STATE_NORMAL (below PBFT_BOOT_MIN_PEERS, C14) |
| H15 | Y-5 re-key with a message in flight | message sent under the old key just before re-gen still verifies within PBFT_Y5_GRACE_MS=5000 (dual-key grace, G9/B11); consensus not paused |

### 3.3 Run command

```bash
cd tools/host_harness
./build_and_run.sh --scenario H2 --seed 42
# Output: scenario report (latency P50/P99, throughput, fork count)
```

---

## 4. T3 — Single-node smoke (on-device)

The simplest on-device test: one ESP32-C3 devkit running the example app.

```c
// test/smoke/main/test_smoke.c
TEST_CASE("single-node smoke: submit and self-commit", "[smoke]") {
    pbft_init(&cfg);
    pbft_register_commit_cb(on_test_commit, NULL);
    pbft_start();

    pbft_tx_t tx = {.payload = (uint8_t*)"hi", .payload_len = 2, .flags = 0};
    TEST_ASSERT_EQUAL(ESP_OK, pbft_submit(&tx));

    // Wait up to 5 s for commit
    for (int i = 0; i < 5000 / 100; i++) {
        vTaskDelay(pdMS_TO_TICKS(100));
        if (g_test_commit_count > 0) break;
    }
    TEST_ASSERT_EQUAL(1, g_test_commit_count);
    pbft_stop();
    pbft_deinit();
}
```

Run via `idf.py -p /dev/ttyUSB0 test` (Unity on-device runner).

**Expected result:** ~5 s test time, PASS.

---

## 5. T4 — Cluster integration (7-node)

Seven ESP32-C3 devkits are flashed with the same firmware, each with a unique `node_id` (0..6) stored in NVS.

### 5.1 Physical setup

```
   ┌──────────────────────────────────────────────┐
   │  Power: 7 × USB (or shared 5 V / 2 A supply) │
   │  Wi-Fi: all on same channel (e.g., ch 6)     │
   │  Distance: 1-3 m apart (lab bench)           │
   │  Enclosure: none (open air)                  │
   └──────────────────────────────────────────────┘
```

### 5.2 Test orchestration

- One laptop connects to all 7 devkits via USB
- A Python orchestrator (`tools/cluster_test/`) drives each via `idf.py monitor` and `idf.py test` APIs
- Logs are collected centrally; correlated by sequence number

### 5.3 Cluster tests

| # | Test | Pass criterion |
|---|------|----------------|
| C1 | All 7 boot, no errors | All reach PBFT_STATE_NORMAL within 10 s |
| C2 | Submit from each node in turn | TX committed at all 7 within 200 ms |
| C3 | Sustained 14 TPS for 10 min | All TXs committed; heap stable; **view-changes allowed if cluster re-converges within 3 s** |
| C4 | Power-cycle one node mid-test | Node re-handshakes; catches up; resumes consensus |
| C5 | Sustained 1 TPS for 24 hours | All TXs committed; no memory leaks (heap_min ≥ 16 KB) |
| C6 | Wi-Fi AP outage (toggle) | STAs reconnect; consensus resumes within 5 s |
| C7 | ESP-NOW channel switch | Nodes fail to communicate; manually restored |

### 5.4 Run command

```bash
cd tools/cluster_test
./run_cluster_test.sh --test C3 --duration 600
# Output: per-node log, central metrics, pass/fail
```

---

## 6. T5 — Byzantine fault injection

The hardest tier. We need to **deliberately make nodes Byzantine** to verify the protocol's safety guarantees.

### 6.1 Byzantine node emulator

`tools/byzantium/` runs on a Raspberry Pi (or laptop with 7 ESP32s over USB) and:

1. Receives all packets via ESP-NOW sniffer (a separate ESP32-C3 in promiscuous mode)
2. Sends crafted packets to selected nodes via their own ESP-NOW interface

### 6.2 Fault modes

| # | Mode | Description | Test target |
|---|------|-------------|-------------|
| BF1 | Equivocation | Send different Prepare to different subsets | quorum math (audit B + verification) |
| BF2 | Replay | Resend captured Prepare after 5 s | anti-replay (CRYPTO §5.2) |
| BF3 | Bad HMAC | Random MAC on Prepare | drop + alarm |
| BF4 | Stale view | Send Prepare from "view v-1" | drop + alarm |
| BF5 | Primary silent | Primary skips Pre-Prepare for 3 TXs | view-change trigger |
| BF6 | Conflicting digest | Send Pre-Prepare(10, A) then (10, B) | digest disagreement → view-change |
| BF7 | Cascade | All 6 Byzantine replicas attack simultaneously | safety MUST hold |
| BF8 | Prepare drop / loss recovery | Selectively drop one replica's Prepare for a seq | the waiting node rebroadcasts on `timeout_ms / 2`, up to `PBFT_MAX_RETRIES`=3 (`retry_count`); if quorum still not reached after 3 retries it triggers a view-change (G8/B8) |

### 6.3 Safety verification

For each Byzantine scenario:

1. Capture full cluster log (every sent/received packet at every node)
2. Compute the **commit set** at every node: set of `(seq, digest)` pairs that reached COMMITTED
3. Assert: **all honest nodes have identical commit sets**
4. Assert: **no two honest nodes commit different digests at the same sequence**

This is the **safety invariant** of PBFT. A single fork is a test failure.

### 6.4 Liveness verification

For each scenario:

1. After scenario ends, submit 100 more TXs
2. Assert: all 100 commit within 30 s
3. If fails: liveness violated (or scenario made cluster unrecoverable — both are bugs)

### 6.5 Retransmit / packet-loss recovery (BF8)

BF8 needs assertions on the per-entry `retry_count` counter, not just the final outcome:

1. Drop exactly one replica's Prepare for a chosen `seq`; let virtual time advance.
2. Assert the waiting node rebroadcasts its Prepare each `timeout_ms / 2` and increments
   `retry_count` once per rebroadcast, capping at `PBFT_MAX_RETRIES`=3 (no 4th rebroadcast).
3. **Recovery path:** if the dropped Prepare is then allowed through before the 3rd retry,
   the seq commits and `retry_count` stops advancing.
4. **Escalation path:** if quorum is still not reached after `retry_count == PBFT_MAX_RETRIES`,
   assert the node triggers a view-change (PBFT_STATE_VIEW_CHANGE) and the cluster resumes
   under the new primary (G8/B8).

### 6.6 Run command

```bash
cd tools/byzantium
./run_byzantine.sh --scenario BF7 --duration 300
# Output: per-node log, safety-check report, liveness-check report
```

---

## 7. T6 — Soak test (24 hours)

Goal: catch memory leaks, heap fragmentation, Y-5 re-gen bugs, NVS wear-out.

### 7.1 Setup

- 7-node cluster, 14 TPS sustained
- Each node logs: `heap_free_bytes`, `heap_min_free_bytes`, `s_metrics` snapshot every 60 s
- NVS write count tracked (one write per `pbft_stop`)

### 7.2 Pass criteria

| Metric | Threshold |
|--------|-----------|
| `heap_min_free_bytes` | ≥ 16 KB at all times |
| Y-5 re-gen completed | exactly 7 (one per node, once per 24 h) |
| TXs committed | 14 TPS × 86400 s = 1,209,600 (all committed) |
| View-changes triggered | ≤ 10 (some are expected from RF noise) |
| Heap fragmentation ratio | < 50% (computed via `heap_caps_get_min_free_size`) |
| NVS wear-out | < 100 writes per namespace (well under 100K limit) |

### 7.3 Output

A 24-hour log file (~10 MB compressed). Trended plots via `tools/plot_soak.py`.

---

## 8. T7 — Performance benchmarks

### 8.1 Latency

Measure end-to-end: from `pbft_submit()` returning to local commit callback firing.

| Metric | Target (ESP-NOW) | Target (Wi-Fi UDP + PS_MIN_MODEM) | Stretch |
|--------|------------------|------------------------------------|---------|
| P50 latency | ≤ 35 ms | ≤ 50 ms | ≤ 25 ms |
| P99 latency | ≤ 100 ms | ≤ 150 ms | ≤ 70 ms |
| P99.9 latency | ≤ 500 ms | ≤ 700 ms | ≤ 300 ms |

### 8.2 Throughput

Sustained TXs/sec, measured at the application callback rate.

| Workload | ESP-NOW target | Wi-Fi UDP target |
|----------|----------------|------------------|
| Max throughput (back-to-back submit) | ≥ 14 TPS | ≥ 9 TPS |
| Sustained (15 min) | ≥ 13 TPS | ≥ 8 TPS |

### 8.3 Heap peak

Per POWER.md §4.4 and MEMORY.md §5.5:

| Phase | Heap peak |
|-------|-----------|
| Boot | ≤ 8 KB transient |
| Y-5 re-gen | ≤ 3 KB transient |
| Per-message | ≤ 1 KB transient |
| Heap min during 24 h soak | ≥ 16 KB |

### 8.4 Power

From POWER.md §4.4:

| Workload | Average current (ESP-NOW) | Average current (Wi-Fi UDP + PS_MIN_MODEM) |
|----------|---------------------------|---------------------------------------------|
| 14 TPS sustained | ~8 mA | ~60 mA |
| 1 TPS sustained | ~2 mA | ~25 mA |

### 8.5 Statistical pass criteria (AUDIT-FIX IR-7, 2026-06-29)

The latency targets in §8.1 and throughput targets in §8.2 are MEANINGFUL only
with a defined statistical methodology. Previously the targets stood alone
with no sample size, warm-up, number of runs, or pass rule — the CI gate was
undefined and would either ship flaky builds or brick CI forever. The table
below is the explicit contract:

| Performance metric | Sample size | Warm-up | Runs | Pass rule |
|--------------------|------------:|--------:|-----:|-----------|
| P50 commit latency (single TX) | ≥ 1000 TXs (random gap) | 60 s | ≥ 10 | P50 ≤ 35 ms in 9 of 10 runs |
| P99 commit latency (single TX) | ≥ 1000 TXs | 60 s | ≥ 10 | P99 ≤ 100 ms in 9 of 10 runs |
| P99.9 commit latency (single TX) | ≥ 1000 TXs | 60 s | ≥ 10 | P99.9 ≤ 500 ms in 9 of 10 runs |
| Sustained throughput (back-to-back) | ≥ 5000 TXs | 60 s | ≥ 3 | ≥ 13 TPS (ESP-NOW) in 2 of 3 runs |
| Heap-min during 24 h soak | 1 sample / 60 s | n/a | 1 | heap_min ≥ 16 KB at every sample |
| View-changes triggered during C3/C5 | runs = 10 min each | n/a | 1 | re-converge < 3 s after each (cluster must apply New-View and resume) |
| Liveness after Byzantine injection (BF1–BF7) | per scenario | 5 s | 1 | cluster commits ≥ 100 TXs within 30 s post-attack |

**Run-to-run independence:** each run is a fresh `idf.py flash` + power-cycle;
no state is shared between runs. The host harness `mock_timer` resets the
virtual time at run boundary.

**Byzantine loss-budget (§5.3 C3 was previously "no view-changes"):** a
correct view-change triggered by a Byzantine primary is NOT a test failure.
The pass criterion is that the cluster re-converges within `3 s` of the
view-change trigger; the new primary applies the New-View and resumes
committing. View-changes from primary misbehavior that fail to re-converge
within `3 s` are a liveness bug.

**TX-loss budget:** at 14 TPS, ≤ 0.01% TX loss due to RF is acceptable (≤ 8
TX/80000). TXs lost during a view-change retried by the application do NOT
count against this budget if the retry commits within `viewchange_timeout_ms`.

---

## 9. CI gate (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: esp-pbft CI

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install ESP-IDF v6.0.1
        run: |
          git clone --depth 1 --branch v6.0.1 https://github.com/espressif/esp-idf.git
          $ESP_IDF/install.sh
      - name: Run unit tests
        run: |
          source $ESP_IDF/export.sh
          cd test
          idf.py --target esp32c3 build
          idf.py --target esp32c3 test
      - name: Coverage check
        run: |
          ./tools/check_coverage.sh --min 80

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run cppcheck
        run: cppcheck --error-exitcode=1 --enable=all src/ include/
      - name: Run clang-tidy
        run: |
          source $ESP_IDF/export.sh
          clang-tidy --warnings-as-errors='*' src/*.c

  host-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build host harness
        run: |
          cd tools/host_harness
          make
      - name: Run H1, H2, H4, H8
        run: |
          for scenario in H1 H2 H4 H8; do
            ./build_and_run.sh --scenario $scenario --seed 42
          done
```

**Gate:** PR merge requires all checks pass.

---

## 10. Determinism and reproducibility

- All host-harness scenarios take a `--seed` flag (default 42)
- All randomness in esp-pbft uses a per-thread `pcg32_random_r()` (deterministic from seed)
- Y-5 stagger offsets are derived from `node_id` (deterministic)
- Cluster logs include `s_view_change_started_ms` for replay

A recorded log can be replayed offline:

```bash
./replay_log.sh cluster_log_2026_06_27.bin
# Output: re-simulated packet flow with safety checks
```

This catches **regressions** — if a code change causes a different outcome for the same seed, the test fails.

---

## 11. Test data and fixtures

```
test/
├── unit/
│   ├── test_config.c
│   ├── test_crypto.c
│   ├── test_consensus.c
│   └── ... (one per module)
├── smoke/
│   ├── main/
│   │   ├── test_smoke.c
│   │   └── CMakeLists.txt
│   └── sdkconfig.defaults
└── fixtures/
    ├── valid_packet.bin          # known-good PBFT Pre-Prepare
    ├── bad_hmac_packet.bin       # corrupted MAC
    ├── stale_view_packet.bin     # old view number
    ├── long_payload.bin          # 256 B payload
    └── replay_attack_packet.bin  # captured + replayed

tools/
├── host_harness/                  # T2 host integration
├── cluster_test/                  # T4 cluster orchestration
├── byzantium/                     # T5 fault injection
└── plot_soak.py                   # T6 visualisation
```

---

## 12. Coverage targets

| Module | Line target | Branch target |
|--------|-------------|---------------|
| `pbft_crypto.c` | 90% | 85% |
| `pbft_consensus.c` | 85% | 80% |
| `pbft_viewchange.c` | 85% | 80% |
| `pbft_checkpoint.c` | 85% | 80% |
| `pbft_network_*.c` | 80% | 75% |
| `pbft_storage.c` | 70% | 65% |
| **Overall** | **80%** | **75%** |

`cppcheck` and `gcc -fanalyzer` must report zero defects on the merged code.

---

## 13. Open questions (test level)

| # | Question | Recommendation |
|---|----------|----------------|
| T1 | How do we test the bootloader/key provisioning path? | Separate integration test in DEPLOYMENT.md. |
| T2 | How do we simulate Y-5 without waiting 24 h? | `pbft_y5_force_regen()` debug API (see API-REFERENCE §14 API4). |
| T3 | How do we simulate ESP-NOW packet loss deterministically? | Use a programmable attenuator (e.g., `tools/attenuator.py` controlling a Hamamitsu attenuator). |
| T4 | Should we have a "release candidate" cluster that runs 24/7? | Yes — for v1.1; out of scope for v1.0. |

---

## 14. References

- [HANDOVER.md](./HANDOVER.md) §6 — test plan TBD (now documented)
- [MEMORY.md](./MEMORY.md) §9 — heap-peak CI gate
- [POWER.md](./POWER.md) §4.4 — performance targets
- [CRYPTO.md](./CRYPTO.md) §8 — per-op latency for crypto budget
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — verification of fixes (A1-A7, B1-B11)
- ESP-IDF Unity docs: `docs/en/contribute/esp-idf-unit-testing-guide.rst`

---

**End of TEST-PLAN.md (v0.1 draft — covers T1-T7 + CI gate)**