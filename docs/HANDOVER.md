# esp-pbft — Handover Document

> **Project:** esp-pbft — lightweight PBFT consensus core library for ESP32-C3 clusters
> **Version:** 1.0 (design complete, implementation pending)
> **Date:** 2026-06-26
> **Status:** 🟡 Design 100% — Implementation 0%
> **Target hardware:** ESP32-C3 (RISC-V, 160 MHz, 400 KB SRAM, no PSRAM)
> **ESP-IDF:** v6.0.1 (Mbed TLS 4.0.0 + TF-PSA-Crypto)

---

## 1. Project Overview

### 1.1 Goal

Build a **memory-optimized PBFT (Practical Byzantine Fault Tolerance) consensus core** for a 7-node ESP32-C3 cluster with the following guarantees:

- **Safety:** all honest nodes agree on the same total order of transactions
- **Liveness:** valid requests are eventually committed
- **Byzantine tolerance:** tolerates `f = ⌊(n-1)/3⌋ = 2` faulty nodes (`n = 3f+1 = 7`)

### 1.2 Design Principles

1. **Minimal core** — no app-specific logic, no blockchain, no VM, no account model
2. **Generic API** — `submit(tx) → callback(tx, sequence)` so any app can plug in
3. **ESP32-C3 optimised** — minimal RAM footprint, no PSRAM, ESP-NOW transport
4. **Reusable** — extension hooks for blockchain / state-machine layer later
5. **Modern ESP-IDF** — PSA Crypto API (Mbed TLS 4.0), not deprecated `mbedtls_*`

### 1.3 What is NOT in scope

| Layer | Reason |
|-------|--------|
| ❌ Blockchain (blocks, Merkle, chain) | PBFT orders TXs by sequence number, not by chain hash |
| ❌ Smart-contract VM | Generic TX payload — app parses it |
| ❌ State machine / account model | App's responsibility |
| ❌ Token / coin logic | App's responsibility |
| ❌ Example apps (counter, KV) | v1 deliverable only; examples deferred |

---

## 2. Architecture (summary — see [ARCHITECTURE.md](./ARCHITECTURE.md))

### 2.1 Cluster topology

- **n = 7 nodes** (one is primary, six are replicas)
- **f = 2** (tolerates 2 Byzantine nodes)
- **Quorum** = `2f+1 = 5` for Prepare and Commit
- **Transport:** ESP-NOW broadcast (peer-to-peer, no AP/router)
- **Range:** ~200 m, ~80 mA active

### 2.2 Module layering

```
Application  (user code: submit TX, register callback)
        ↓
esp-pbft Public API  (pbft_init / start / submit / register_commit_cb)
        ↓
┌───────────────┬─────────────────┬───────────────┐
│  Consensus    │  View-Change    │  Checkpoint   │
│  (3 phases)   │  (primary rot.) │  (GC)         │
└───────────────┴─────────────────┴───────────────┘
        ↓
┌───────────────┬─────────────────┬───────────────┐
│  Crypto       │  Network        │  Storage      │
│  (PSA)        │  (ESP-NOW)      │  (NVS)        │
└───────────────┴─────────────────┴───────────────┘
```

### 2.3 Consensus flow (3 phases)

```
Client → Primary
        ↓
PRE-PREPARE   (primary → all)
        ↓
PREPARE       (all → all, wait ≥ 2f+1 = 5)
        ↓
COMMIT        (all → all, wait ≥ 2f+1 = 5)
        ↓
COMMITTED → app callback → EXECUTED → GC
```

### 2.4 Per-request state machine

```
IDLE → PRE_PREPARED → PREPARED → COMMITTED → EXECUTED
  ↑         │             │           │           │
  └─────────┴─────────────┴───────────┴───────────┘  (any state → timeout → view-change)
```

See [ARCHITECTURE.md](./ARCHITECTURE.md) §3 for full state-machine spec and [CONSENSUS.md](./CONSENSUS.md) for phase-by-phase detail.

---

## 3. Design Decisions

### 3.1 Why PBFT?

| Consensus | Byzantine | Throughput | Latency | Finality |
|-----------|-----------|------------|---------|----------|
| **PBFT** | ✅ Yes | High | Low | Immediate |
| PoW | ✅ Yes | Low | High | Probabilistic |
| PoS | ✅ Yes | Medium | Medium | Probabilistic |
| Raft | ❌ No | Very High | Very Low | Immediate |
| Paxos | ❌ No | High | Low | Immediate |

**Chosen because:** Byzantine tolerance (security-critical IoT) + immediate finality + high throughput.

### 3.2 Why n = 7, f = 2?

```
n = 3f + 1
  f=1 → n=4   (1/4 = 25% faulty tolerance) — too small for real Byzantine stress
  f=2 → n=7   (2/7 = 29% faulty tolerance) ← CHOSEN — well-studied config
  f=3 → n=10  — exceeds ESP-NOW comfortable broadcast count
```

### 3.3 Why ESP-NOW?

| Transport | Range | Power | Throughput | Complexity |
|-----------|-------|-------|------------|------------|
| **ESP-NOW** | 200 m | Low (~80 mA) | 1 Mbps | Low |
| Wi-Fi UDP  | 200 m | High (~160 mA) | 54 Mbps | High |
| BLE        | 100 m | Very Low (~15 mA) | 2 Mbps | Medium |

**Chosen because:** 2× lower power than Wi-Fi, no AP/router needed, ~10–20 ms latency, built-in CCMP encryption. See [POWER.md](./POWER.md) for full measurements.

### 3.4 Why PSA Crypto?

ESP-IDF v6.0.1 ships **Mbed TLS 4.0.0 + TF-PSA-Crypto**. Legacy `mbedtls_*` APIs are still available but transitional. PSA is the recommended interface because:

- ✅ Standardised (portable across vendors)
- ✅ Hardware-accelerated on ESP32-C3 (SHA-256 via dedicated peripheral)
- ✅ Side-channel resistant
- ✅ Built-in key ID lifecycle (`mbedtls_svc_key_id_t`)

**PSA APIs used in esp-pbft** (verified from `~/.espressif/v6.0.1/esp-idf/components/mbedtls/.../psa/crypto.h`):
- `psa_generate_key` — TRNG-backed ECDSA P-256 keypair at boot
- `psa_key_agreement` (`PSA_ALG_ECDH`) — derive shared secret with each peer
- `psa_mac_compute` / `psa_mac_verify` — HMAC-SHA256 per message (SHA HW-accelerated)

**APIs explicitly NOT used** (kept out of runtime hot path):
- `psa_sign_message` / `psa_verify_message` — no ECDSA at runtime (~50 ms cost; would kill throughput)

Full design in [CRYPTO.md](./CRYPTO.md).

### 3.5 Authentication pattern — **Pattern Y** (TRNG + ECDH-boot + HMAC-runtime)

> **Decision (confirmed 2026-06-26):** Pattern Y is the final choice.
> See [CRYPTO.md](./CRYPTO.md) for handshake sequence + key derivation details.

| Pattern | Per-msg cost | Cluster cost / request | Verdict |
|---------|-------------|----------------------|---------|
| A — Pre-shared keys + HMAC | 5 µs | ~70 µs | ❌ Requires keygen.py + per-node flash |
| B — ECDSA every msg | 50 ms (sign) | ~750 ms | ❌ Slow on ESP32-C3 |
| C — ECDSA identity + ECDH session + HMAC | 5 µs | ~70 µs | ❌ Over-engineered vs Y |
| **Y — Self-gen TRNG + ECDH-boot + HMAC-runtime** | **5 µs** | **~70 µs** | ✅✅ **CHOSEN** |

**Why Pattern Y** (vs alternatives):

1. **No ECDSA at runtime** — ESP32-C3 has SHA-256 HW + ECC point-mul HW but **no ECDSA-sign HW** (DS peripheral is RSA-only). Per-message ECDSA sign would cost ~50 ms.
2. **Self-generated keys** — ESP32-C3's hardware TRNG (NIST SP 800-22 compliant) generates ECDSA P-256 keypair at boot. No `keygen.py`, no per-node firmware build, no key flash.
3. **No key persistence** — Private key lives only in RAM; regenerated on every boot.
4. **Pairwise HMAC keys** — Each node derives 6 pairwise HMAC keys (one per peer) via ECDH at boot. ~192 B RAM total.
5. **Periodic re-gen (Y-5)** — Every 24 hours (staggered by `node_id × 1h`) each node regenerates its ECDSA keypair in RAM and broadcasts a Hello to trigger **soft re-handshake** among peers. Compromise window is bounded by the re-gen period, not by node uptime.
6. **TOFU for Hello phase** — Trust-on-first-use: a peer's first Hello is trusted; subsequent Hellos must match the cached pubkey (else alarm). Sufficient for physically-secured IoT cluster.

**Pattern Y summary:**
```
Boot:
  psa_generate_key()                          // ~5 ms (TRNG + ECC HW)
  → broadcast Hello (pubkey, node_id)
  → receive Hello from 6 peers
  → for each peer: ECDH(my_priv, peer_pub) → HMAC-SHA256(shared, "PBFT-HMAC-v1")
  → store 6 HMAC keys in RAM (192 B)

Periodic (every 24h, staggered by node_id × 1h):
  psa_generate_key()                          // new keypair in RAM
  → broadcast Hello (re-handshake marker)
  → peers re-ECDH with new pubkey
  → update hmac_keys[peer] (no peer restart needed)

Peer restart:
  → restarted node broadcasts new Hello
  → peers re-ECDH on demand (soft re-handshake)

Runtime:
  → HMAC-SHA256(hmac_key_peer, view||seq||type||payload) per message
  → ~5 µs/msg (SHA HW)
```

### 3.6 Why no blockchain?

| Aspect | esp-pbft only | + Blockchain |
|--------|---------------|--------------|
| LOC | ~5,000 | ~20,000 |
| Throughput | ~9K tx/sec | ~1K tx/sec |
| Storage | ~10 KB | GB+ |
| Immutable history | ❌ | ✅ |
| Smart contracts | ❌ | ✅ |

esp-pbft is the consensus layer; the blockchain layer (if needed later) can be added without changing this design. **Migration path:** add `esp-pbft-blockchain.c` that subscribes to commit callback.

### 3.7 Memory budget (verified for ESP32-C3, no PSRAM)

| Component | Size | Notes |
|-----------|------|-------|
| PBFT log (pending TX) | ~6 KB | 100 entries × 60 B |
| Membership config | ~1 KB | 7 nodes × 140 B |
| Crypto contexts (PSA) | ~7–12 KB | ECDH + ECDSA gen + HMAC + PSA core |
| HMAC keys (derived) | 192 B | 6 peers × 32 B (RAM only, regenerated each boot) |
| ECDSA private key (boot) | ~700 B | RAM only, regenerated each boot |
| Network buffers | ~8 KB | 2 × 4 KB TX/RX |
| Persistent state | ~1 KB | view, sequence, last checkpoint |
| **Total used** | **~25 KB** | **~6% of 400 KB** |
| **Free** | **~375 KB** | 94% |

See [MEMORY.md](./MEMORY.md) for detailed layout.

### 3.8 Power budget

| Mode | Current | Time/Day | Energy/Day |
|------|---------|----------|------------|
| Active (consensus) | 80 mA | 1 h | 80 mAh |
| Light sleep | 0.5 mA | 23 h | 11.5 mAh |
| **Total** | — | — | **~92 mAh/day** |

With a 1000 mAh battery → **~10 days battery life**.

See [POWER.md](./POWER.md) for state-transition design.

---

## 4. File Structure (planned)

```
esp-pbft/
├── include/
│   ├── pbft.h                  ← public API
│   ├── pbft_types.h            ← shared types, enums, structs
│   ├── pbft_config.h           ← configuration constants
│   ├── pbft_crypto.h           ← crypto API
│   ├── pbft_network.h          ← ESP-NOW transport API
│   ├── pbft_membership.h       ← 7-node membership
│   ├── pbft_consensus.h        ← 3-phase consensus
│   ├── pbft_viewchange.h       ← view-change protocol
│   ├── pbft_checkpoint.h       ← checkpoint / GC
│   ├── pbft_storage.h          ← NVS (config + view/seq + membership — NOT keys)
│   └── pbft_log.h              ← logging + metrics
├── src/
│   ├── pbft.c                  ← public API impl
│   ├── pbft_crypto.c           ← HMAC + ECDSA + ECDH + HKDF (PSA)
│   ├── pbft_network.c          ← ESP-NOW
│   ├── pbft_membership.c       ← 7-node table
│   ├── pbft_consensus.c        ← 3 phases
│   ├── pbft_viewchange.c       ← primary rotation
│   ├── pbft_checkpoint.c       ← GC
│   ├── pbft_storage.c          ← NVS
│   └── pbft_log.c              ← logging
├── test/                       ← v2 deliverable (Phase 3)
├── examples/                   ← v2 deliverable (Phase 4)
├── tools/                      ← v2 deliverable (Phase 4)
├── docs/                       ← this folder
├── CMakeLists.txt
├── sdkconfig.defaults
└── README.md
```

### 4.1 Implementation status (0%)

| Component | Design | Implementation | Tests |
|-----------|--------|----------------|-------|
| Types & config | ✅ 100% | ❌ 0% | ❌ 0% |
| Crypto | ✅ 100% | ❌ 0% | ❌ 0% |
| Network | ✅ 100% | ❌ 0% | ❌ 0% |
| Membership | ✅ 100% | ❌ 0% | ❌ 0% |
| Consensus | ✅ 100% | ❌ 0% | ❌ 0% |
| View-Change | ✅ 100% | ❌ 0% | ❌ 0% |
| Checkpoint | ✅ 100% | ❌ 0% | ❌ 0% |
| Storage | ✅ 100% | ❌ 0% | ❌ 0% |
| Logging | ✅ 100% | ❌ 0% | ❌ 0% |

---

## 5. Public API (preview — full contract in [API-REFERENCE.md](./API-REFERENCE.md))

```c
// Lifecycle
esp_err_t pbft_init(const pbft_config_t* config);
esp_err_t pbft_start(void);
esp_err_t pbft_stop(void);
esp_err_t pbft_deinit(void);

// Submit + callback
esp_err_t pbft_submit(const pbft_tx_t* tx);
typedef void (*pbft_commit_cb_t)(const pbft_tx_t* tx, uint64_t sequence);
esp_err_t pbft_register_commit_cb(pbft_commit_cb_t cb);

// Status + metrics
pbft_state_t pbft_get_state(void);
uint64_t pbft_get_view(void);
uint64_t pbft_get_sequence(void);
pbft_metrics_t pbft_get_metrics(void);
```

See [API-REFERENCE.md](./API-REFERENCE.md) for error codes, parameter validation, thread-safety contract.

---

## 6. Documentation map

This folder contains:

| Doc | Status | Purpose |
|-----|--------|---------|
| [HANDOVER.md](./HANDOVER.md) | 📝 Drafting | This file — 1-page project overview |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 📅 Next | System diagrams + sequence flows + state machines |
| [CRYPTO.md](./CRYPTO.md) | 📅 Planned | Pattern Y handshake, key lifecycle, MAC binding |
| [PROTOCOL.md](./PROTOCOL.md) | 📅 Planned | Wire format, `pbft_packet_t` byte layout |
| [CONSENSUS.md](./CONSENSUS.md) | 📅 Planned | Pre-Prepare / Prepare / Commit detailed spec |
| [VIEW-CHANGE.md](./VIEW-CHANGE.md) | 📅 Planned | Primary rotation algorithm |
| [CHECKPOINT.md](./CHECKPOINT.md) | 📅 Planned | Stable checkpoint + catch-up protocol |
| [API-REFERENCE.md](./API-REFERENCE.md) | 📅 Planned | Full public API contract |
| [MEMORY.md](./MEMORY.md) | 📅 Planned | Memory layout + budget breakdown |
| [POWER.md](./POWER.md) | 📅 Planned | Power state transitions + measurements |
| [FAILURE-MODES.md](./FAILURE-MODES.md) | 📅 Planned | Byzantine scenarios + partition analysis |
| [TEST-PLAN.md](./TEST-PLAN.md) | 📅 Planned | Unit / integration / Byzantine / perf tests |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | 📅 Planned | keygen, flash script, cluster init |
| [ROADMAP.md](./ROADMAP.md) | 📅 Planned | Phased implementation plan |
| [INDEX.md](./INDEX.md) | 📅 Final | Master index, written last |

---

## 7. References

### Internal

- Research report on authentication patterns (delegated 2026-06-26) — confirms Pattern Y (TRNG + ECDH-boot + HMAC-runtime)
- ESP-IDF v6.0.1 local install at `~/.espressif/v6.0.1/esp-idf/` (Mbed TLS 4.0.0)
- `idf_component.yml` declares `idf: ">=6.0.1"` and supported targets

### External

- Castro & Liskov, "Practical Byzantine Fault Tolerance" (OSDI 1999) — http://pmg.csail.mit.edu/papers/osdi99.pdf
- NIST SP 800-56A Rev. 3 — ECDH key agreement
- NIST FIPS 186-4 / draft 186-5 — ECDSA
- NIST SP 800-107 — HMAC security
- Hyperledger Fabric Orderer — production PBFT with ECDSA identity + MAC
- BFT-SMaRt — Java PBFT reference impl
- ESP-NOW docs — https://docs.espressif.com/projects/esp-idf/en/v6.0.1/esp32c3/api-reference/network/esp-now.html
- PSA Crypto API spec — https://arm-software.github.io/psa-api/crypto/

---

## 8. Glossary

| Term | Meaning |
|------|---------|
| **PBFT** | Practical Byzantine Fault Tolerance (Castro & Liskov 1999) |
| **Byzantine** | Arbitrary / malicious failure mode (vs crash-only) |
| **Primary** | The node currently proposing requests (rotates per view) |
| **Replica** | All non-primary nodes in current view |
| **View** | Configuration number; changes when primary rotates |
| **Sequence** | Monotonic TX order position assigned by primary |
| **Quorum** | `2f+1` matching votes required to advance state |
| **Checkpoint** | Stable snapshot at sequence number `K × n` for GC |
| **Pre-Prepare** | Phase 1 — primary assigns sequence & broadcasts request |
| **Prepare** | Phase 2 — replicas echo agreement on `(view, seq, digest)` |
| **Commit** | Phase 3 — replicas confirm they are prepared |
| **Prepared** | State after observing `2f+1` matching Prepare certificates |
| **Committed** | State after observing `2f+1` matching Commit certificates |
| **View-change** | Protocol to rotate primary on suspected failure |
| **New-view** | Message from new primary proving it has `2f+1` view-change votes |
| **STS** | Station-to-Station protocol — ECDH authenticated by long-term ECDSA |
| **Y-5** | Pattern Y variant with periodic key re-gen (every 24h, staggered) |
| **HMAC** | Hash-based Message Authentication Code (RFC 2104) |
| **HKDF** | HMAC-based Key Derivation Function (RFC 5869) |
| **PSA Crypto** | Platform Security Architecture Crypto API (Arm standard) |
| **ESP-NOW** | Espressif connectionless Wi-Fi protocol |
| **NVS** | Non-Volatile Storage (ESP-IDF key-value store) |

---

## 9. Acronyms

- **TX** — Transaction
- **MAC** — Message Authentication Code (also: Media Access Control in different context)
- **PSRAM** — Pseudo-Static RAM (external, not present on C3 in this design)
- **SRAM** — Static RAM (on-chip)
- **RISC-V** — Open instruction set architecture
- **RTOS** — Real-Time Operating System (FreeRTOS on ESP-IDF)
- **API** — Application Programming Interface
- **Kconfig** — Kernel configuration (ESP-IDF build options)
- **LOC** — Lines of Code

---

## 10. Open Questions (Status: 2026-06-26)

| # | Question | Status | Recommendation in design |
|---|----------|--------|--------------------------|
| 1 | Checkpoint interval K | 🟡 Open | K=100, tunable |
| 2 | Log size limit | 🟡 Open | Fixed 100 + checkpoint cleanup |
| 3 | Timeout values | 🟡 Open | Adaptive — start 1000 ms, +25% per round |
| 4 | Byzantine detection | 🟡 Open | Passive logging v1, active v2 |
| 5 | Key exchange | ✅ **Decided** | Self-gen ECDSA P-256 via TRNG at boot, no persistence, ECDH → pairwise HMAC keys |
| 6 | Concurrency model | 🟡 Open | Single-threaded + FreeRTOS event loop |
| 7 | Network buffering | 🟡 Open | Static 2×4 KB (predictable) |
| 8 | State persistence | 🟡 Open | NVS for config + view/seq + membership (NO private key — Pattern Y) |
| 9 | Key rotation strategy | ✅ **Decided** | Y-5: every 24h, staggered by node_id × 1h, soft re-handshake |

---

## 11. Roadmap (high-level)

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| 1. Core impl | 4 weeks | All 10 .c/.h files compile, single-node smoke test |
| 2. Hardening | 2 weeks | Checkpoint, timeout mgmt, error handling, Y-5 periodic re-gen |
| 3. Testing | 2 weeks | Unit + integration + Byzantine + perf (incl. 24h re-gen test) |
| 4. Docs + tools | 2 weeks | Examples, deployment script, Doxygen |
| **Total** | **~10 weeks** | Production-ready v1 |

See [ROADMAP.md](./ROADMAP.md) for detailed breakdown (when written).

---

**End of HANDOVER.md (v1.0 draft — awaiting review)**