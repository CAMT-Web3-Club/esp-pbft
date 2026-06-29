# esp-pbft — Handover Document

> **Project:** esp-pbft — lightweight PBFT consensus core library for ESP32-C3 clusters
> **Version:** 1.0 (design complete, implementation pending)
> **Date:** 2026-06-27
> **Status:** ✅ Design complete — see [INDEX.md](./INDEX.md) for the full document map
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
3. **ESP32-C3 optimised** — minimal RAM footprint, no PSRAM, transport configurable at compile time (ESP-NOW default, Wi-Fi UDP optional)
4. **Reusable** — extension hooks for blockchain / state-machine layer later
5. **Transport-agnostic** — same PBFT wire format works on ESP-NOW or Wi-Fi UDP multicast; swap via Kconfig
6. **Modern ESP-IDF** — PSA Crypto API (Mbed TLS 4.0), not deprecated `mbedtls_*`
7. **Strict C style** — follow Zephyr MISRA-C subset + CERT C + ESP-IDF style (see [CODING_STANDARD.md](./CODING_STANDARD.md))

### 1.3 What is NOT in scope

| Layer | Reason |
|-------|--------|
| ❌ Blockchain (blocks, Merkle, chain) | PBFT orders TXs by sequence number, not by chain hash |
| ❌ Smart-contract VM | Generic TX payload — app parses it |
| ❌ State machine / account model | App's responsibility |
| ❌ Token / coin logic | App's responsibility |
| ❌ Example apps (counter, KV) | v1 deliverable only; examples deferred |

### 1.4 Coding standards (summary)

Full detail in [CODING_STANDARD.md](./CODING_STANDARD.md). Top-3 ranked:

| Rank | Adopted from | Why |
|------|--------------|-----|
| **1** | **Zephyr MISRA-C 2012 subset** (147 rules) | Modern, embedded-focused, open source |
| **2** | **CERT C** | Security for crypto + memory + string safety |
| **3** | **Linux kernel** (`goto cleanup`, no Hungarian) + **BSD KNF** macro hygiene | Proven patterns, readable |

**Key conventions** (full list in §CODING_STANDARD):
- C standard: **gnu23 (C23)** — supported by GCC 15.2 + ESP-IDF preferred
- Indent: 4 spaces (ESP-IDF default, no tabs)
- Line length: ≤120 chars
- Header guards: `#pragma once` (ESP-IDF preferred)
- Function size: ≤80 lines, ≤7 local variables
- Naming: `pbft_` prefix for public, `s_` prefix for static (ESP-IDF style)
- Types: use `<stdint.h>` (`uint32_t`, `uint64_t`) — never plain `int` for sizes/counters
- Buffer types: use `uint8_t` for byte buffers — never plain `char`
- Constants: decimal only — never octal (`0777` → typo)

**Static analysis (CI gate)**:
- `gcc -fanalyzer` (built-in, no install)
- `cppcheck 2.17.1` (apt)
- `clang-tidy 19` (apt)
- `astyle 3.4.7` (formatter — match ESP-IDF style)

**Anti-patterns banned (top 5 of 20)**:
- A1: `malloc`/`calloc`/`realloc` in any PBFT handler (already enforced by §11 static memory)
- A2: Ignoring `mbedtls_*` / `psa_*` return values
- A3: `sprintf`/`printf` with non-literal format strings (RCE surface!)
- A4: Pointer aliasing casts `(uint32_t *)bytes`
- A6: Plain `int` for sizes/sequences — use `uint64_t` for view/sequence

**Safe additional compiler flags** (component-private, do NOT conflict with ESP-IDF):
```cmake
target_compile_options(${COMPONENT_LIB} PRIVATE
    -Wpedantic -Wshadow=local -Wstrict-prototypes
    -Wmissing-prototypes -Wpointer-arith -Wcast-align
    -Wnull-dereference -Wdouble-promotion -Wformat=2 -Wundef
)
```

**AVOID** (conflict with ESP-IDF): `-Wconversion`, `-Wold-style-definition`, `-Wstrict-aliasing=3`

---

## 2. Architecture (summary — see [ARCHITECTURE.md](./ARCHITECTURE.md))

### 2.1 Cluster topology

- **n = 7 nodes** (one is primary, six are replicas)
- **f = 2** (tolerates 2 Byzantine nodes)
- **Quorum** = `2f+1 = 5` for Prepare and Commit
- **Transport:** **configurable at compile time** via Kconfig
  - `CONFIG_PBFT_TRANSPORT_ESP_NOW=y` (default): peer-to-peer, no AP, ~80 mA
  - `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y`: AP-based, IGMP multicast, ~160 mA
  - See [PROTOCOL.md](./PROTOCOL.md) for transport abstraction details

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
  f=3 → n=10  — exceeds comfortable broadcast count (ESP-NOW officially ≤20 peers)
```

### 3.3 Transport selection (compile-time)

| Transport | Range | Power | Throughput | Cluster size | AP needed | Multicast |
|-----------|-------|-------|------------|--------------|-----------|-----------|
| **ESP-NOW** (default) | 200 m | Low (~80 mA) | 1 Mbps | ≤10 (officially ≤20) | ❌ | Broadcast to FF:FF:FF:FF:FF:FF |
| **Wi-Fi UDP** | 200 m | High (~160 mA) | 54 Mbps | unlimited | ✅ | IGMP group join |

**Default = ESP-NOW** (`CONFIG_PBFT_TRANSPORT_ESP_NOW=y`) because:
- ✅ 2× lower power than Wi-Fi (battery life)
- ✅ No AP/router needed (simpler deployment)
- ✅ ~10–20 ms latency (good enough for PBFT)
- ✅ Built-in CCMP encryption
- ✅ Sufficient for n=7 cluster

**Select Wi-Fi UDP** (`CONFIG_PBFT_TRANSPORT_WIFI_UDP=y`) when:
- Larger cluster (>10 nodes) needed
- AP infrastructure already exists
- Higher throughput required (>1 Mbps)

**Constraint:** Both transports use the same PBFT wire format (see [PROTOCOL.md](./PROTOCOL.md)). Switching transport does NOT require changing application code.

See [POWER.md](./POWER.md) for ESP-NOW measurements and [FAILURE-MODES.md](./FAILURE-MODES.md) for transport-specific failure modes (e.g., IGMP group leave, broadcast storm).

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
| Throughput | **14 TPS** sustained (no batching; 33 TPS sequential in CONSENSUS §9) | **133 TPS** batched (N=4) |
| Storage | ~10 KB | GB+ |
| Immutable history | ❌ | ✅ |
| Smart contracts | ❌ | ✅ |

esp-pbft is the consensus layer; the blockchain layer (if needed later) can be added without changing this design. **Migration path:** add `esp-pbft-blockchain.c` that subscribes to commit callback.

### 3.7 Memory budget (verified for ESP32-C3, no PSRAM)

**Reconciled 2026-06-27** (corrects earlier conflicting figures — see [DESIGN-AUDIT.md §A5](./DESIGN-AUDIT.md)).

| Component | Size | Notes |
|-----------|------|-------|
| PBFT log (pending TX) | **~36 KB** | 100 entries × ~360 B (inline 256 B payload + state) |
| Membership config | ~1 KB | 7 nodes × ~140 B |
| Crypto contexts (PSA) | ~3 KB | ECDH + ECDSA gen + HMAC + PSA core |
| HMAC keys (derived) | ~280 B | 6 peers × ~32 B + handles |
| ECDSA private key (boot) | ~32 B | RAM only, regenerated each boot |
| Peer pubkey cache (TOFU) | ~487 B | 6 peers × 65 B (0x04 ‖ X_BE ‖ Y_BE) + PSA key handles |
| Network buffers | ~16 KB | 4 KB TX + 4 KB RX + 8 KB arena |
| Dedup cache (audit B10) | ~10 KB | 256 executed digests |
| TX queue (for `submit_from_isr`) | ~4.5 KB | 16 entries × ~280 B |
| View state + V-set arena | ~5 KB | single view state + 7 × 488 B VC slots |
| Checkpoint proofs | ~640 B | 8 in-flight proofs |
| Misc (metrics, config, etc.) | ~260 B | — |
| **Subtotal esp-pbft source (BSS)** | **~72 KB** | ~18% of 400 KB |
| | | |
| ESP-IDF Wi-Fi driver | ~15-25 KB | depends on transport |
| ESP-NOW component | ~5 KB | always linked |
| FreeRTOS core | ~10 KB | — |
| PSA crypto core | ~10 KB | — |
| lwIP (excluded for ESP-NOW build via `CONFIG_LWIP_ENABLE=n`) | 0 (excluded) / ~45 KB (Wi-Fi UDP) | |
| Other ESP-IDF | ~5 KB | |
| **Subtotal ESP-IDF (ESP-NOW build, no lwIP)** | **~45 KB** | |
| **Subtotal ESP-IDF (Wi-Fi UDP build)** | **~100 KB** | |
| | | |
| **Grand total (ESP-NOW build)** | **~117 KB** | **~29% of 400 KB** |
| **Grand total (Wi-Fi UDP build)** | **~177 KB** | **~44% of 400 KB** |
| **Free (ESP-NOW)** | **~283 KB** | 71% |
| **Free (Wi-Fi UDP)** | **~223 KB** | 56% |

See [MEMORY.md](./MEMORY.md) for the detailed per-module layout, `static_assert` catalog, and NVS schema.

### 3.8 Power budget (per transport)

**ESP-NOW build (default) — superseded by [POWER.md §4.4 workload model + §12.6 corrected budget]:**

| Mode | Current | Time/Day | Energy/Day |
|------|---------|----------|------------|
| Active (consensus) | see POWER.md §12.6 | derived (workload-model) | see POWER.md §12.6 |
| Light sleep | 0.13 mA (130 µA datasheet typ) | per POWER.md §4.2 | derived |
| **Total** | ~7.96 mA avg | — | **~191 mAh/day** (from POWER.md §4.4) |

→ 1000 mAh battery → **~5 days battery life** at 14 TPS sustained (light-sleep on)
→ 1 TPS sustained → **~40 days** (POWER.md §4.5)
→ Lighter workload: see POWER.md §4.4 for the full model

|

| Mode | Current | Time/Day | Energy/Day |
|------|---------|----------|------------|
| Active (consensus) | see POWER.md §12.6 | derived (workload-model) | see POWER.md §12.6 |
| Light sleep | 0.13 mA (130 µA datasheet typ) | per POWER.md §4.2 | derived |
| **Total** | ~7.96 mA avg | — | **~191 mAh/day** (from POWER.md §4.4) |

→ 1000 mAh battery → **~5 days battery life** at 14 TPS sustained (light-sleep on)
→ 1 TPS sustained → **~40 days** (POWER.md §4.5)
→ Lighter workload: see POWER.md §4.4 for the full model

|

---

## 4. File Structure (planned)

```
esp-pbft/
├── include/
│   ├── pbft.h                  ← public API
│   ├── pbft_types.h            ← shared types, enums, structs
│   ├── pbft_config.h           ← configuration constants
│   ├── pbft_crypto.h           ← crypto API
│   ├── pbft_network.h          ← transport abstraction interface
│   ├── pbft_network_espnow.c   ← ESP-NOW impl (CONFIG_PBFT_TRANSPORT_ESP_NOW)
│   ├── pbft_network_wifi_udp.c ← Wi-Fi UDP multicast impl (CONFIG_PBFT_TRANSPORT_WIFI_UDP)
│   ├── pbft_membership.h       ← 7-node membership
│   ├── pbft_consensus.h        ← 3-phase consensus
│   ├── pbft_viewchange.h       ← view-change protocol
│   ├── pbft_checkpoint.h       ← checkpoint / GC
│   ├── pbft_storage.h          ← NVS (config + view/seq + membership — NOT keys)
│   └── pbft_log.h              ← logging + metrics
├── src/
│   ├── pbft.c                  ← public API impl
│   ├── pbft_crypto.c           ← HMAC + ECDSA + ECDH + HKDF (PSA)
│   ├── pbft_network.c          ← transport abstraction (selects ESP-NOW or Wi-Fi UDP at compile time)
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

### 4.1 Implementation status

**Design:** complete across all 17 documents.
**Implementation:** pending (Phase 1 of [ROADMAP.md](./ROADMAP.md) not started).

| Component | High-level | Detailed | Implementation | Tests |
|-----------|-----------|----------|----------------|-------|
| Types & config | ✅ In HANDOVER | ✅ API-REFERENCE §2 | ❌ 0% | ❌ 0% |
| Crypto | ✅ In HANDOVER | ✅ CRYPTO.md | ❌ 0% | ❌ 0% |
| Network | ✅ In HANDOVER | ✅ PROTOCOL.md | ❌ 0% | ❌ 0% |
| Membership | ✅ In HANDOVER | ✅ API-REFERENCE + MEMORY | ❌ 0% | ❌ 0% |
| Consensus | ✅ In HANDOVER | ✅ CONSENSUS.md | ❌ 0% | ❌ 0% |
| View-Change | ✅ In HANDOVER | ✅ VIEW-CHANGE.md | ❌ 0% | ❌ 0% |
| Checkpoint | ✅ In HANDOVER | ✅ CHECKPOINT.md | ❌ 0% | ❌ 0% |
| Storage | ✅ In HANDOVER | ✅ MEMORY.md §4 | ❌ 0% | ❌ 0% |
| Logging | ✅ In HANDOVER | ✅ API-REFERENCE §4.5 | ❌ 0% | ❌ 0% |
| Power mgmt | ✅ In HANDOVER | ✅ POWER.md | ❌ 0% | ❌ 0% |
| Failure handling | ✅ In HANDOVER | ✅ FAILURE-MODES.md | ❌ 0% | ❌ 0% |

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

This folder contains 17 design documents totalling ~340 KB. See [INDEX.md](./INDEX.md) for the full index.

| Doc | Status | Purpose |
|-----|--------|---------|
| [HANDOVER.md](./HANDOVER.md) | ✅ Done | This file — 1-page project overview |
| [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) | ✅ Done | Cross-doc consistency check (2026-06-27) |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | ✅ Done | System diagrams + sequence flows + state machines |
| [CRYPTO.md](./CRYPTO.md) | ✅ Done | Pattern Y handshake, key lifecycle, MAC binding |
| [PROTOCOL.md](./PROTOCOL.md) | ✅ Done | Wire format, `pbft_packet_t` byte layout |
| [CONSENSUS.md](./CONSENSUS.md) | ✅ Done | Pre-Prepare / Prepare / Commit detailed spec |
| [VIEW-CHANGE.md](./VIEW-CHANGE.md) | ✅ Done | Primary rotation, V-set/O-set, New-View verification |
| [CHECKPOINT.md](./CHECKPOINT.md) | ✅ Done | Stable checkpoint + GC + state transfer |
| [API-REFERENCE.md](./API-REFERENCE.md) | ✅ Done | Full public C API contract + error codes |
| [MEMORY.md](./MEMORY.md) | ✅ Done | Per-module layout + static_assert catalog |
| [POWER.md](./POWER.md) | ✅ Done | Power state machine + Wi-Fi PS modem mode |
| [FAILURE-MODES.md](./FAILURE-MODES.md) | ✅ Done | Byzantine scenarios + recovery playbook |
| [TEST-PLAN.md](./TEST-PLAN.md) | ✅ Done | Unit / integration / Byzantine / perf / soak tests |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | ✅ Done | NVS provisioning + OTA + Y-5 runbook |
| [ROADMAP.md](./ROADMAP.md) | ✅ Done | Phased implementation plan (P1-P5) |
| [INDEX.md](./INDEX.md) | ✅ Done | Master index |
| [CODING_STANDARD.md](./CODING_STANDARD.md) | ✅ Done | Zephyr MISRA-C subset + CERT C + ESP-IDF style (47 KB) |

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
| **Kconfig** | ESP-IDF compile-time configuration system (`CONFIG_PBFT_TRANSPORT_ESP_NOW` / `..._WIFI_UDP`) |
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