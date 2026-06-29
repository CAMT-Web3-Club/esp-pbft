# esp-pbft — Architecture

> **Status:** 🟡 Draft
> **Scope:** System overview, component layering, sequence diagrams, state machines
> **Out of scope:** Detailed crypto internals → [CRYPTO.md](./CRYPTO.md), wire format → [PROTOCOL.md](./PROTOCOL.md)

---

## 1. System overview

esp-pbft is a **3-layer PBFT consensus core** for a 7-node ESP32-C3 cluster. The library exposes a generic `submit(tx) → callback(tx, sequence)` API and is transport-agnostic (ESP-NOW or Wi-Fi UDP multicast, compile-time select).

### 1.1 Cluster topology

```
              7-node cluster (f = 2, quorum = 5)
   ┌──────────────────────────────────────────────────────┐
   │                                                      │
   │   Node 0 (primary)         ◀──  rotating  ──▶        │
   │   ─── broadcast Pre-Prepare                          │
   │   ─── collect Prepare (≥2f+1 = 5)                    │
   │   ─── collect Commit (≥2f+1 = 5)                     │
   │   ─── fire commit_callback(seq, tx)                  │
   │                                                      │
   │   Node 1..6 (replicas)     ◀──  monitor primary  ──▶ │
   │   ─── receive Pre-Prepare                            │
   │   ─── send Prepare                                   │
   │   ─── send Commit                                    │
   │   ─── apply TX to app on commit                      │
   │                                                      │
   └──────────────────────────────────────────────────────┘
                          │
                          │ transport (compile-time Kconfig)
                          ▼
        ┌──────────────────────────────────────┐
        │  ESP-NOW (default)  |  Wi-Fi UDP     │
        │  peer-to-peer       |  AP + IGMP     │
        │  80 mA, no AP       |  160 mA, AP    │
        └──────────────────────────────────────┘
```

### 1.2 Layered architecture (3 layers)

```
┌─────────────────────────────────────────────────────────┐
│ Layer 3: APPLICATION (user code)                          │
│   ── submit(tx)                                           │
│   ── register commit_callback(tx, sequence)              │
│   ── get state / view / sequence / metrics               │
└────────────────────────┬────────────────────────────────┘
                         │ public API (pbft.h)
┌────────────────────────┴────────────────────────────────┐
│ Layer 2: CONSENSUS CORE (public + internal modules)      │
│                                                         │
│  pbft_consensus.c   ←── 3-phase state machine            │
│  pbft_viewchange.c  ←── primary rotation                 │
│  pbft_checkpoint.c  ←── stable checkpoint + GC           │
│  pbft_log.c         ←── in-memory log (last 100 TXs)     │
└────────────────────────┬────────────────────────────────┘
                         │ internal API
┌────────────────────────┴────────────────────────────────┐
│ Layer 1: SERVICES                                         │
│                                                         │
│  pbft_crypto.c       ←── Pattern Y: TRNG + ECDH + HMAC   │
│  pbft_network.c      ←── transport abstraction            │
│    ├── pbft_network_espnow.c                             │
│    └── pbft_network_wifi_udp.c                           │
│  pbft_storage.c      ←── NVS (config, no keys)           │
│  pbft_membership.c   ←── 7-node table + peer cache       │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Boot sequence

### 2.1 First-time boot (cold start)

```
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │ Node X   │         │ Node Y   │         │ Node Z   │
   └────┬─────┘         └────┬─────┘         └────┬─────┘
        │                    │                    │
   t=0  │ TRNG keygen        │                    │
        │ (5 ms)             │                    │
        ▼                    │                    │
   ECDSA P-256 keypair       │                    │
   (my_priv, my_pub)         │                    │
        │                    │                    │
        │ random delay       │                    │
        │ 0-3s (staggered)   │                    │
        │                    │                    │
   t≈1  │ broadcast Hello    │                    │
        │ (my_pub, node_id)  │                    │
        ├───────────────────▶│ broadcast Hello    │
        │                    ├───────────────────▶│
        │                    │                    │
        │ cache pubkey       │ cache pubkey       │
        │ (TOFU: first wins) │ (TOFU: first wins) │
        │                    │                    │
        │ ECDH(my_priv,      │ ECDH(my_priv,      │
        │   peer_pub_Y)      │   peer_pub_X)      │
        │   = shared_XY      │   = shared_YX      │
        │                    │                    │
        │ HMAC-SHA256(       │ HMAC-SHA256(       │
        │   shared_XY,       │   shared_YX,       │
        │   "PBFT-HMAC-v1")  │   "PBFT-HMAC-v1")  │
        │   = hmac_XY        │   = hmac_YX        │
        │                    │                    │
        ▼                    ▼                    ▼
   Store 6 HMAC keys     Store 6 HMAC keys    Store 6 HMAC keys
   in RAM (192 B)        in RAM (192 B)       in RAM (192 B)
        │                    │                    │
   t≈3  │ PBFT ready          │ PBFT ready         │ PBFT ready
        │                    │                    │
```

### 2.2 Boot mitigation

- **Staggered boot** — random 0–3s delay before first Hello broadcast
- **Why:** 7 nodes booting simultaneously → 7 simultaneous broadcasts → ~21 redundant Hellos per receiver → risk of ESP-NOW broadcast storm
- **Trade-off:** cluster ready in ≤3s + handshake (worst case ≈ 5s)

### 2.3 Soft re-handshake (Pattern Y-5, every 24h)

```
   t=0h      Node 0 boots normally
             ...
   t=24h     Node 0 timer fires (its offset = 0)
             ─────────────────────────────────────
   t=24h+ε   Node 0 generates new ECDSA keypair
             (TRNG, 5 ms, RAM only)
                  │
                  ▼
             broadcast Hello (new_pub, node_id=0)
             + re-handshake marker in payload
                  │
                  ├──────────▶  Node 1: cache new pubkey
                  ├──────────▶  Node 2: cache new pubkey
                  │            ...
                  └──────────▶  Node 6: cache new pubkey
                                  │
                                  ▼
                            Each peer:
                            ECDH(my_priv, new_pub_0) = new_shared
                            HMAC-SHA256(new_shared, "PBFT-HMAC-v1")
                              = new_hmac[0]
                                  │
                                  ▼
                            Update hmac_keys[0] in RAM
                                  │
                                  ▼
                            Re-handshake complete (~50 ms)
```

**Compromise window:** bounded by re-gen period (24h), not by node uptime.

**Note:** Self re-gen and peer re-gen are independent. Node 0's re-gen triggers peers to re-ECDH with node 0; it does NOT trigger peers to re-ECDH with each other.

---

## 3. Consensus sequence (steady state)

### 3.1 Normal request flow (primary honest)

```
   Client    Primary (p)    Replicas (1..6)    App
     │            │                │           │
     │ submit(tx) │                │           │
     ├───────────▶│                │           │
     │            │ Pre-Prepare    │           │
     │            │ (v, n, d, tx)  │           │
     │            ├───────────────▶│           │
     │            │                │ Verify HMAC
     │            │                │ Accept    │
     │            │                │           │
     │            │ Prepare        │           │
     │            │ (v, n, d)      │           │
     │            │◀───────────────┤           │
     │            ├───────────────▶│           │
     │            │  ... (all nodes broadcast)  │
     │            │                │           │
     │            │ collect Prepare (≥2f+1=5)  │
     │            │                │           │
     │            │ state → PREPARED (own)     │
     │            │                │           │
     │            │ Commit         │           │
     │            │ (v, n, d)      │           │
     │            │◀───────────────┤           │
     │            ├───────────────▶│           │
     │            │  ... (all nodes broadcast)  │
     │            │                │           │
     │            │ collect Commit (≥2f+1=5)   │
     │            │                │           │
     │            │ state → COMMITTED (own)    │
     │            │                │ state → COMMITTED
     │            │                │ commit_callback(tx, n)
     │            │                ├──────────▶│
     │            │                │           │ apply TX
     │            │                │           │
```

### 3.2 Message sequence numbers

| Field | Size | Purpose |
|-------|------|---------|
| `view` | u32 | Which primary is current leader |
| `sequence` | u64 | Monotonic TX order (assigned by primary) |
| `digest` | 32 B | SHA-256 of TX payload (request id) |
| `mac` | 32 B | HMAC-SHA256 over `(view ‖ seq ‖ type ‖ digest ‖ payload)` |

### 3.3 State machine (per request, per replica)

```
                         receive Pre-Prepare
                                 │
                                 ▼
              ┌──────────────────────────────────┐
              │  IDLE                            │
              └──────────────┬───────────────────┘
                             │ verify HMAC + accept Pre-Prepare
                             │ → send Prepare
                             ▼
              ┌──────────────────────────────────┐
              │  PRE_PREPARED                    │
              └──────────────┬───────────────────┘
                             │ collect ≥2f+1 matching Prepare
                             ▼
              ┌──────────────────────────────────┐
              │  PREPARED                        │
              └──────────────┬───────────────────┘
                             │ → send Commit
                             │ (already in PREPARED locally,
                             │  just send Commit broadcast)
                             ▼
              ┌──────────────────────────────────┐
              │  PREPARED + sent Commit          │
              └──────────────┬───────────────────┘
                             │ collect ≥2f+1 matching Commit
                             ▼
              ┌──────────────────────────────────┐
              │  COMMITTED                       │
              │  → fire commit_callback(tx, n)   │
              └──────────────┬───────────────────┘
                             │ apply to app + log entry
                             ▼
              ┌──────────────────────────────────┐
              │  EXECUTED                        │
              │  → mark for checkpoint cleanup   │
              └──────────────────────────────────┘

   Transition on view-change (any state):
     PRE_PREPARED / PREPARED → reset to IDLE (new view)
     COMMITTED / EXECUTED → KEEP (carries over to new view)
```

### 3.4 Primary's state machine (simpler)

```
   receive submit(tx) from app
                │
                ▼
   assign sequence = next_seq()
                │
                ▼
   broadcast Pre-Prepare(v, sequence, digest, tx)
                │
                ▼
   collect ≥2f+1 Prepare (own + replicas)
                │
                ▼
   broadcast Commit(v, sequence, digest)
                │
                ▼
   collect ≥2f+1 Commit
                │
                ▼
   fire commit_callback(tx, sequence)
                │
                ▼
   next_seq++
```

---

## 4. Component dependencies

```
pbft.c (public API)
  ├── pbft_consensus.c
  │     ├── pbft_log.c
  │     ├── pbft_crypto.c ── ESP-IDF mbedtls
  │     ├── pbft_network.c ─┐
  │     │                    ├── pbft_network_espnow.c (if CONFIG_PBFT_TRANSPORT_ESP_NOW)
  │     │                    └── pbft_network_wifi_udp.c (if CONFIG_PBFT_TRANSPORT_WIFI_UDP)
  │     ├── pbft_membership.c
  │     └── pbft_storage.c ── ESP-IDF NVS
  ├── pbft_viewchange.c
  │     └── pbft_consensus.c (read state)
  ├── pbft_checkpoint.c
  │     └── pbft_log.c (cleanup)
  └── pbft_log.c
```

**Dependency rule:** no cycles. Public API depends on consensus, consensus depends on services (crypto, network, storage).

---

## 5. Data flow (steady state, single request)

```
                        ┌────────────────────────────┐
                        │        pbft.c               │
                        │   pbft_submit(tx) called    │
                        └─────────────┬──────────────┘
                                      │ enqueue TX
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_consensus.c         │
                        │  if primary:               │
                        │   assign seq, build PrePrep │
                        │  build signed packet       │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_crypto.c            │
                        │   psa_mac_compute(         │
                        │     key=hmac_keys[peer],   │
                        │     input=view||seq||...   │
                        │   ) → 32-byte MAC          │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_network.c           │
                        │   send(broadcast, packet)   │
                        │   → ESP-NOW | Wi-Fi UDP    │
                        └─────────────┬──────────────┘
                                      │
                                      │ UDP packet (broadcast)
                                      ▼
                                [7 ESP32-C3]
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_network.c (rx)      │
                        │   on packet received        │
                        │   dispatch to consensus     │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_crypto.c            │
                        │   psa_mac_verify(...)       │
                        │   accept or reject         │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    pbft_consensus.c         │
                        │   state machine:           │
                        │   Pre-Prepare → Prepare    │
                        │   → Commit → Callback      │
                        └─────────────┬──────────────┘
                                      │
                                      ▼
                        ┌────────────────────────────┐
                        │    Application              │
                        │   commit_callback(tx, seq) │
                        │   → apply to app state     │
                        └────────────────────────────┘
```

---

## 6. Concurrency model

- **Single-threaded** for v1 (FreeRTOS task = `pbft_task`, priority 5)
- **Event loop** driven by ESP-IDF `esp_event`:
  - Network RX events → dispatch to consensus
  - Timer events (view-change timeout, Y-5 re-gen) → dispatch to relevant module
  - App `submit()` calls → enqueue to consensus TX queue
- **No locks** (single-threaded = no shared-state races)
- **Trade-off:** simpler code, no priority inversion risk; max throughput limited by single-core CPU

See [API-REFERENCE.md](./API-REFERENCE.md) §thread-safety for exact contract.

---

## 7. View-change (preview)

When a replica suspects the primary is faulty (Prepare timeout, or `2f+1` ViewChange messages received):

```
   Replica R detects primary fault
                │
                ▼
   broadcast VIEW-CHANGE(v+1, R, last_stable_checkpoint_proof)
                │
                ▼
   collect ≥2f+1 VIEW-CHANGE messages for v+1
                │
                ▼
   New primary p_new = (v+1) mod n
                │
                ▼
   p_new builds NEW-VIEW(v+1, V-set, O-set)
   - V-set = set of received VIEW-CHANGE messages
   - O-set = Pre-Prepare messages to re-propose (from V-set's checkpoints)
                │
                ▼
   broadcast NEW-VIEW(v+1)
                │
                ▼
   Replicas verify NEW-VIEW (≥2f+1 VIEW-CHANGE proofs)
                │
                ▼
   State: view = v+1, p_new becomes primary
   Resume consensus from O-set
```

Full details in [VIEW-CHANGE.md](./VIEW-CHANGE.md) (TBD).

---

## 8. Checkpoint (preview)

Every `K` sequences (e.g., K=100), replicas broadcast CHECKPOINT messages with state digest:

```
   seq reaches K * n
                │
                ▼
   broadcast CHECKPOINT(seq, state_digest, app_state_hash)
                │
                ▼
   collect ≥2f+1 matching CHECKPOINT
                │
                ▼
   stable checkpoint = seq
   → garbage-collect log entries below seq
   → discard Pre-Prepare/Prepare below seq
```

Full details in [CHECKPOINT.md](./CHECKPOINT.md) (TBD).

---

## 9. Open design questions (architectural level)

These are higher-level than the HANDOVER.md open questions:

| # | Question | Why it matters | Status |
|---|----------|----------------|--------|
| A1 | Should primary's Prepare phase be skipped? (optimisation) | Reduces 1 round-trip in normal case | 🟡 Open |
| A2 | Should we batch multiple submit()s into one Pre-Prepare? | Throughput vs latency trade-off | 🟡 Open |
| A3 | How do we handle client retry on timeout? (PBFT doesn't define this) | Liveness under packet loss | 🟡 Open |
| A4 | Should view-change include app state proof? | Catch-up cost vs safety | 🟡 Open |

---

## 10. References

- [HANDOVER.md](./HANDOVER.md) — decisions summary (transport, Pattern Y, n=7)
- [CRYPTO.md](./CRYPTO.md) — Pattern Y handshake + HMAC details
- [PROTOCOL.md](./PROTOCOL.md) — wire format + transport abstraction
- [CONSENSUS.md](./CONSENSUS.md) — detailed 3-phase spec
- [VIEW-CHANGE.md](./VIEW-CHANGE.md) — primary rotation
- [CHECKPOINT.md](./CHECKPOINT.md) — garbage collection
- Castro & Liskov, "Practical Byzantine Fault Tolerance" (OSDI 1999)

---

## 11. Static memory discipline

**Principle:** esp-pbft uses **only static allocation** for all internal state. No `malloc`, `calloc`, `realloc`, or `free` calls in the hot path. All buffers, contexts, and data structures are compile-time-sized and live in BSS.

### 11.1 Why static only

| Benefit | Impact |
|---------|--------|
| **Deterministic boot** | Memory available at start = always same |
| **No heap fragmentation** | Long-running cluster (months) won't degrade |
| **Predictable budget** | Compile-time check that total ≤ 400 KB SRAM |
| **No ENOMEM at runtime** | Hot path never blocked by allocation failure |
| **Easier safety review** | All memory visible at compile time |
| **Better for OTA** | Memory layout stable across versions |

### 11.2 Static allocation inventory — 4-region layout (TinyBFT §VI-A)

esp-pbft organizes all static state into **4 regions** by lifetime, inspired by
[TinyBFT §VI-A](https://doi.org/10.1109/RTAS61025.2024.00026). This separation:

- Makes memory budget explicit per region (each fits a `static_assert`)
- Enables selective NVM migration in v2 (Agreement+Checkpoint → SPI Flash)
- Simplifies static analysis (lifetime-based reasoning)

| Region | Data | Size | Lifetime | Default | v2 NVM? |
|--------|------|------|----------|---------|---------|
| **1. Agreement** | `pbft_log[]` (100 entries × ~360 B) | **~36 KB** | Per-view, GC at checkpoint | BSS | SPI Flash |
| | V-set arena (7 × 248 B) | ~1.7 KB | Per-view-change | BSS | SPI Flash |
| **2. Checkpoint** | Checkpoint proof table | ~640 B | Stable (new = discard old) | BSS | SPI Flash |
| | State digests (K+1 entries) | ~256 B | Stable | BSS | SPI Flash |
| **3. Event** | New-View inbox (1 slot) | ~1.3 KB | One-shot per view-change | BSS | RAM |
| | Catch-up request inbox (7 slots) | ~560 B | One-shot per peer | BSS | RAM |
| | Catch-up response inbox (7 slots) | ~1.4 KB | One-shot per peer | BSS | RAM |
| **4. Scratch** | MAC input buffer (48 + 256 B) | **304 B** | Reused per message | **BSS** (was stack) | RAM |
| | Network TX/RX arena | 8 KB | Reused per send/recv | BSS | RAM |
| | TX queue (ISR submit) | ~4.5 KB | Reusable ring | BSS | RAM |
| **Cross-region** | `pbft_config_t` (active config) | ~100 B | Permanent | BSS | RAM |
| | ECDSA private key (current boot) | 32 B | RAM, regenerated on Y-5 | BSS | RAM |
| | ECDSA public key (own) | 65 B | RAM, regenerated on Y-5 | BSS | RAM |
| | Peer pubkey cache (7 × 65 B) | 455 B | Permanent (TOFU) | BSS | RAM |
| | HMAC keys (7 × 32 B) | 224 B | RAM, regenerated on Y-5 | BSS | RAM |
| | Dedup / executed-digest cache (256 × 40 B) | ~10 KB | Permanent (FIFO) | BSS | RAM |
| | PSA crypto contexts (ECDSA, ECDH, HMAC) | ~3 KB | Permanent | BSS | RAM |
| | Metrics counters | ~200 B | Permanent | BSS | RAM |
| | **Total static (esp-pbft source)** | **~72 KB** | **BSS** | — | BSS | mixed |

Arrays are sized `[7]` to the cluster size (self slot included/unused); 6 peers are
actually used. **Grand total ~117 KB** (lwIP excluded) including the ESP-IDF /
FreeRTOS / PSA library footprint — see [MEMORY.md](./MEMORY.md) §3 for the full
breakdown. esp-pbft source contains **zero `malloc`**; PSA crypto and FreeRTOS
allocate internally (§11.5).

> **Why Scratch moved to BSS (Q2 fix)?** A 304 B stack array + 4 KB TX buffer + 1.3 KB
> New-View + etc. on the call stack = real risk of overflow on the 8 KB FreeRTOS
> task stack. The Scratch region is shared (one writer at a time per task) and
> zeroed between uses, so BSS is the right home.

### 11.3 Compile-time bounds checking

Use `static_assert()` (C11) or `BUILD_BUG_ON()` to enforce:

```c
// Compile-time checks (verified by idf.py build)
_Static_assert(sizeof(pbft_node_state_t) <= PBFT_NODE_STATE_MAX_BYTES,
               "pbft_node_state_t exceeded budget");

_Static_assert(PBFT_LOG_MAX_ENTRIES == 100,
               "PBFT_LOG_MAX_ENTRIES must be 100 for memory budget");

_Static_assert(PBFT_CLUSTER_SIZE == 7,
               "PBFT_CLUSTER_SIZE must be 7 (n=7, f=2)");
```

If any check fails, **build fails** (not runtime crash).

### 11.4 Stack allocation (still allowed)

Function-local variables are fine — they live on the stack and are bounded by `CONFIG_ESP_MAIN_TASK_STACK_SIZE` or per-task stack size:

```c
esp_err_t pbft_verify_hmac(const uint8_t* key, const uint8_t* data, size_t len) {
    uint8_t calc_mac[32];                  // OK: 32 B on stack
    psa_mac_compute(..., calc_mac, ...);
    return memcmp(calc_mac, ..., 32) == 0 ? ESP_OK : ESP_FAIL;
    // No malloc, no static state, safe to call concurrently within the same task
}
```

### 11.5 What is NOT static (acceptable exceptions)

| Item | Why dynamic is OK |
|------|-------------------|
| **NVS blob read/write** | ESP-IDF NVS internally allocates; we just call `nvs_get_blob` (caller-supplied buffer is static) |
| **ESP-NOW peer registration** | ESP-IDF manages internally; we call `esp_now_add_peer` once at boot |
| **Wi-Fi driver internal buffers** | Managed by ESP-IDF component, not esp-pbft code |
| **PSA crypto internal scratch** | Managed by mbedtls core; sized via `MBEDTLS_PSA_{KEY,ALGO}_BITS` macros (compile-time) |
| **mbedtls SSL contexts** | Only used during TLS handshake (not in our pattern; handshake uses PSA key agreement directly) |

These are **library-internal allocations** outside our control. We never call `malloc`/`free` directly in esp-pbft.

### 11.6 Consequences for design

| Design choice | Why static requires it |
|---------------|------------------------|
| **Fixed cluster size** | `PBFT_CLUSTER_SIZE = 7` at compile time |
| **Fixed log size** | `PBFT_LOG_MAX_ENTRIES = 100` at compile time |
| **No dynamic node join** | Cannot allocate peer slot at runtime; new node must be in cluster config (compile-time) |
| **Fixed payload size** | TX payload ≤ 256 B (compile-time max) |
| **No string interning** | All messages use fixed-size byte buffers, not heap-allocated strings |

### 11.7 Compile-time configuration (Kconfig)

```kconfig
# sdkconfig.defaults additions
CONFIG_PBFT_LOG_MAX_ENTRIES=100
CONFIG_PBFT_CLUSTER_SIZE=7
CONFIG_PBFT_TX_PAYLOAD_MAX=256
CONFIG_PBFT_TRANSPORT_ESP_NOW=y
# CONFIG_PBFT_TRANSPORT_WIFI_UDP is not set
```

Each option drives a `static_assert` in `pbft_config.h`.

### 11.8 Memory audit checklist (before merge)

Every PR must confirm:

- [ ] No `malloc`/`calloc`/`realloc`/`free` in new code (grep verification)
- [ ] No FreeRTOS `xQueueCreate`/`xTaskCreate` with dynamic sizes
- [ ] All new buffers are `static` or BSS
- [ ] `static_assert` added for any new compile-time bounds
- [ ] Total static usage updated in `HANDOVER.md` §3.7 table
- [ ] `idf.py size-components` shows no unexpected growth

### 11.9 Trade-offs accepted

| Trade-off | Mitigation |
|-----------|------------|
| Cannot add node at runtime | Provisioning via NVS at factory; Y-5 re-handshake allows refresh |
| Large payload requires recompile | TX_PAYLOAD_MAX = 256 covers typical IoT use cases |
| Wasted RAM if cluster smaller than 7 | Acceptable — 192 B HMAC keys, ~500 B peer cache total |
| Cannot grow log beyond 100 entries | Checkpoint GC runs every K=100 sequences, keeps bounded |

See [MEMORY.md](./MEMORY.md) (when written) for detailed per-module memory layout.

### 11.10 4-region per-region static_assert

Each region has its own compile-time bound check, so a config regression that
bloats one region will fail the build with a clear message rather than silently
overflowing RAM at runtime. Adopted from TinyBFT §VI-D.

```c
// 1. Agreement region (per-view, GC'd at checkpoint)
_Static_assert(sizeof(s_agreement) <= 40 * 1024,
               "Agreement region exceeds 40 KB budget — adjust PBFT_LOG_MAX_ENTRIES");
_Static_assert(sizeof(s_vc_set_arena) <= 2 * 1024,
               "VC arena exceeds 2 KB budget");

// 2. Checkpoint region (stable)
_Static_assert(sizeof(s_checkpoint) <= 1 * 1024,
               "Checkpoint region exceeds 1 KB budget — adjust K+1");

// 3. Event region (one-shot)
_Static_assert(sizeof(s_event) <= 4 * 1024,
               "Event region exceeds 4 KB budget — adjust inbox sizes");

// 4. Scratch region (reusable; was stack in CRYPTO.md §5.2)
_Static_assert(sizeof(s_scratch) <= 16 * 1024,
               "Scratch region exceeds 16 KB budget — risk of NVM round-trip latency");

// Grand total
_Static_assert(sizeof(s_esp_pbft) <= 80 * 1024,
               "esp-pbft BSS exceeds 80 KB — incompatible with ESP32-C3 400 KB RAM");
```

> **Why per-region asserts?** A single `static_assert(sizeof(s_esp_pbft) <= 80 KB)`
> only fires after multiple regions bloated at once. Per-region asserts localize
> the regression to the offending code, which speeds review and prevents the
> budget from drifting silently across releases. Re-baseline the budgets only
> with a deliberate `MEMORY.md` §3 update + commit message explaining the
> increase (e.g. "Bump Agreement region from 40 KB to 48 KB to support 8-node
> cluster; corresponding HANDOVER §3.7 total updated to ~80 KB").

---

**End of ARCHITECTURE.md (v0.3 draft — 4-region layout from TinyBFT §VI-A added)**