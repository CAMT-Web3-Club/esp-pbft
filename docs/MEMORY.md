# esp-pbft — Memory Layout and Budget

> **Status:** 🟡 Draft (v0.1)
> **Scope:** per-module memory layout, static_assert catalog, NVS layout, peak stack analysis, fragmentation safety, runtime heap tracking.
> **Out of scope:** API → [API-REFERENCE.md](./API-REFERENCE.md); checkpoint details → [CHECKPOINT.md](./CHECKPOINT.md).

This document is the **canonical reference for memory use in esp-pbft**. It supersedes the partial inventory in ARCHITECTURE.md §11 and resolves the contradictions surfaced in [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) (issue **A5**: 60 B vs 120 B vs 280 B per entry).

---

## 1. Inventory principle

esp-pbft source contains **zero `malloc`/`calloc`/`realloc`/`free` calls** in the steady state. Every internal buffer, counter, table, and context lives in BSS at compile time. This section enumerates the complete inventory.

Heap usage from PSA crypto, FreeRTOS, and ESP-IDF components (ESP-NOW, Wi-Fi) is tracked separately in §5.

---

## 2. Per-module layout

### 2.1 `pbft_config_t` (active configuration)

```c
static pbft_config_t s_config;     // ~120 B
```

| Field | Size |
|-------|------|
| `my_node_id` | 1 B |
| `cluster_size` | 1 B |
| `cluster_members` | 4 B (pointer, points into s_cluster_members) |
| `prepare_timeout_ms` | 4 B |
| `commit_timeout_ms` | 4 B |
| `viewchange_timeout_ms` | 4 B |
| `payload_max` | 1 B |
| padding | ~7 B |
| `state_digest_cb` | 4 B |
| `log_cb` | 4 B |
| (other) | ~84 B |
| **Total** | **~120 B** |

### 2.2 `pbft_node_desc_t[7]` (cluster table)

```c
static pbft_node_desc_t s_cluster_members[7];   // 7 × ~16 B = ~112 B
```

### 2.3 PBFT log (resolves audit A5 — inline payload)

```c
static pbft_log_entry_t s_log[PBFT_LOG_MAX_ENTRIES];   // 100 × ~360 B = ~36 KB
```

This is **THE** canonical `pbft_log_entry_t` (single source of truth; CONSENSUS &
CHECKPOINT must match this exactly or reference it — see DECISIONS G4):

```c
typedef enum {
    PBFT_ENTRY_FREE = 0,
    PBFT_ENTRY_PRE_PREPARED,
    PBFT_ENTRY_PREPARED,
    PBFT_ENTRY_COMMITTED,
    PBFT_ENTRY_EXECUTED,
} pbft_entry_state_t;

typedef struct {
    uint64_t           sequence;            //  8 B  assigned by primary
    uint32_t           view;                //  4 B  view it was pre-prepared in
    uint8_t            digest[32];          // 32 B  SHA-256 of request payload
    uint8_t            msg_type;            //  1 B  request type tag
    uint16_t           payload_len;         //  2 B  bytes used in payload[]
    uint8_t            payload[256];        // 256 B inline request (PBFT_TX_PAYLOAD_MAX)
    pbft_entry_state_t state;               //  4 B  FREE/PRE_PREPARED/PREPARED/COMMITTED/EXECUTED
    uint8_t            prepare_bitmask;     //  1 B  bit i = replica i sent Prepare
    uint8_t            commit_bitmask;      //  1 B  bit i = replica i sent Commit
    uint8_t            exec_bitmask;        //  1 B  bit i = replica i confirmed Execute (checkpoint)
    bool               have_pre_prepare;    //  1 B  this entry has a stored Pre-Prepare
    bool               own_prepare_sent;    //  1 B  this node broadcast its Prepare
    bool               own_commit_sent;     //  1 B  this node broadcast its Commit
    bool               is_checkpoint;       //  1 B  sequence is a checkpoint boundary
    uint8_t            retry_count;         //  1 B  retransmits so far (<= PBFT_MAX_RETRIES)
    uint64_t           state_entered_ms;    //  8 B  timestamp of last state transition
    uint64_t           commit_received_ms;  //  8 B  timestamp first Commit observed
    // padding for alignment                // ~16 B
    // ————————————————————————————————————————————————
    // Total                                  ~360 B
} pbft_log_entry_t;
```

Quorum counts come from `__builtin_popcount(entry->prepare_bitmask)` etc. — there are
**no** `prepare_count` / `commit_count` fields and **no** per-entry `app_cb` (the commit
callback is registered once globally, API-REFERENCE §5).

**Compilation check:**

```c
_Static_assert(sizeof(pbft_log_entry_t) <= 384,
               "pbft_log_entry_t exceeded 384 B sub-budget");
_Static_assert(sizeof(pbft_log_entry_t) * PBFT_LOG_MAX_ENTRIES <= 40 * 1024,
               "PBFT log exceeded 40 KB total budget");
```

### 2.4 View state

```c
static pbft_view_state_t s_view_state;   // ~120 B
```

```c
typedef struct {
    uint32_t            view;
    pbft_view_phase_t   phase;
    uint8_t             primary_id;
    uint64_t            view_change_started_ms;
    uint64_t            new_view_started_ms;
    uint64_t            next_sequence;
    uint64_t            last_committed_seq;
    uint64_t            last_executed_seq;
    // NOTE: low/high watermark live in pbft_watermark_state_t (§2.7), NOT here —
    //       view_state must NOT duplicate them (single owner = pbft_watermark_state_t).
    uint8_t             vc_set_count;
    uint8_t             vc_set_from[7];
    const pbft_view_change_t* vc_set[7];
    uint32_t            prepare_timeout_ms;
    uint32_t            commit_timeout_ms;
    uint32_t            viewchange_timeout_ms;
    // —————————————
    // Total: ~120 B
} pbft_view_state_t;
```

### 2.5 V-set arena (VIEW-CHANGE.md §4.1)

```c
static pbft_view_change_t s_vc_set_arena[7];   // 7 × 248 B = ~1.7 KB
                                                  // (each slot sized for max prepared count)
```

Each `pbft_view_change_t` slot holds up to `PBFT_VC_MAX_PREPARED = 4` prepared entries:

```c
slot size = 88 + 4 × 40 = 248 B  // ≤ ESP-NOW 250 B
```

> **Note (no-runtime-fallback):** `PBFT_VC_MAX_PREPARED = 4` (not 10) is enforced at compile time so that
> View-Change fits in ESP-NOW 250 B. If you need to carry more prepared entries, you must select
> `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y` AND use New-View over UDP. View-Change still fits ESP-NOW
> in the default config.

### 2.6 Checkpoint proof table (CHECKPOINT.md §5.3)

```c
static pbft_checkpoint_proof_t s_checkpoint_proofs[8];   // 8 × ~80 B = ~640 B
```

```c
typedef struct {
    uint64_t   sequence;
    uint8_t    digest[32];
    uint8_t    sender_bitmask;
    uint8_t    count;
    bool       is_stable;
    uint8_t    padding[35];
    // ———————
    // Total: ~80 B
} pbft_checkpoint_proof_t;
```

Eight slots support one in-flight checkpoint proof per node plus headroom.

### 2.7 Watermark state

```c
static pbft_watermark_state_t s_watermarks;   // ~50 B
```

**Watermark ownership:** `pbft_watermark_state_t` is the *single* owner of `low_watermark`
and `high_watermark`. `pbft_view_state_t` (§2.4) MUST NOT duplicate these fields.
`high_watermark = low_watermark + PBFT_LOG_MAX_ENTRIES` (CHECKPOINT.md §4 is canonical).

### 2.8 Network buffers

```c
static uint8_t s_tx_buffer[4096];           // 4 KB
static uint8_t s_rx_buffer[4096];           // 4 KB
static uint8_t s_rx_arena[8192];            // 8 KB — holds full PBFT messages for parsing
```

`tx_buffer` is large enough for the largest UDP message (New-View ≤ ~6 KB). RX arena supports two concurrent RX (current + previous).

### 2.9 Crypto state (Pattern Y)

```c
static psa_key_id_t   s_my_priv_key_id;           // 4 B (handle)
static uint8_t        s_my_pub[65];               // 65 B (cached for Hello: 0x04 ‖ X_BE ‖ Y_BE)
static psa_key_id_t   s_peer_pub_key_ids[7];      // 7 × 4 B = 28 B
static uint8_t        s_peer_pub_arena[7][65];    // 7 × 65 B = 455 B (raw bytes)
static psa_key_id_t   s_hmac_key_ids[7];          // 7 × 4 B = 28 B
static uint8_t        s_hmac_key_arena[7][32];    // 7 × 32 B = 224 B
static uint8_t        s_zero_key[32];             // 32 B — sentinel for "no key"
// —————————
// Total: ~828 B
```

`psa_key_attributes_t` is a transient PSA struct (~80 B) used during key creation only — stack-allocated.

### 2.10 Executed-digest cache (audit B10)

```c
#define PBFT_DEDUP_CACHE_SIZE 256
static struct {
    uint8_t  digest[32];
    uint64_t sequence;
} s_dedup_cache[PBFT_DEDUP_CACHE_SIZE];   // 256 × 40 B = ~10 KB
static uint16_t s_dedup_head;             // 1 B — next slot to write
```

Ring buffer; on overflow, the oldest entry is evicted (FIFO).

### 2.11 Membership cache (peer pubkey storage)

Already covered in §2.9 as `s_peer_pub_arena`.

### 2.12 Metrics

```c
static pbft_metrics_t s_metrics;   // ~120 B
```

### 2.13 Pending-TX queue (for `pbft_submit_from_isr`)

```c
static pbft_tx_queue_entry_t s_tx_queue[16];   // 16 × ~280 B = ~4.5 KB
```

Each entry holds a TX (inline payload) + the originating task handle for ack.

### 2.14 Misc state

| Item | Size |
|------|------|
| Discovery state (Hello tracking) | ~50 B |
| Boot state (staggered delay) | ~20 B |
| Alarm/event flags | ~20 B |
| Padding | ~50 B |
| **Total** | **~140 B** |

---

## 3. Total static memory

### 3.1 ESP-NOW build

| Module | Size |
|--------|------|
| Config | ~120 B |
| Cluster members | ~112 B |
| PBFT log (100 × 360 B) | ~36 KB |
| View state | ~120 B |
| V-set arena (7 × 248 B) | ~1.7 KB |
| Checkpoint proofs | ~640 B |
| Watermark state | ~50 B |
| Network buffers | ~16 KB |
| Crypto state | ~828 B |
| Dedup cache | ~10 KB |
| Metrics | ~120 B |
| TX queue | ~4.5 KB |
| Misc | ~140 B |
| **Subtotal (esp-pbft source)** | **~72 KB** |
| | |
| ESP-IDF Wi-Fi driver (lightweight) | ~15 KB |
| ESP-NOW component | ~5 KB |
| FreeRTOS core | ~10 KB |
| PSA crypto core | ~10 KB |
| lwIP (minimal — not used in ESP-NOW build but linked) | ~30 KB* |
| Other ESP-IDF | ~5 KB |
| **Subtotal (ESP-IDF)** | **~75 KB** |
| | |
| **Grand total (ESP-NOW build)** | **~147 KB** |

\* lwIP can be excluded from ESP-NOW build by setting `CONFIG_LWIP_ENABLE=y` to `n` — saves 30 KB. **Recommended for ESP-NOW-only deployment.**

### 3.2 Wi-Fi UDP build

| Module | Size |
|--------|------|
| esp-pbft source (same) | ~72 KB |
| ESP-IDF Wi-Fi driver (full) | ~25 KB |
| lwIP (full stack with IGMP) | ~45 KB |
| DHCP server (for AP) | ~5 KB |
| ESP-NOW component (still linked) | ~5 KB |
| FreeRTOS core | ~10 KB |
| PSA crypto core | ~10 KB |
| Other ESP-IDF | ~5 KB |
| **Subtotal (ESP-IDF)** | **~105 KB** |
| | |
| **Grand total (Wi-Fi UDP build)** | **~177 KB** |

### 3.3 Available on ESP32-C3

- Total SRAM: **400 KB**
- Reserved for bootloader / ISR / FreeRTOS idle: ~30 KB
- **Available for application**: ~370 KB

| Build | esp-pbft usage | % of available | Free |
|-------|---------------|----------------|------|
| ESP-NOW (with lwIP excluded) | ~117 KB | 32% | ~253 KB |
| ESP-NOW (with lwIP) | ~147 KB | 40% | ~223 KB |
| Wi-Fi UDP | ~177 KB | 48% | ~193 KB |

**Conclusion:** all builds fit comfortably within 400 KB SRAM. The original HANDOVER §3.7 claim of "~40 KB total" was off by ~3× — the actual cost is dominated by the inline-payload PBFT log.

---

## 4. NVS layout

NVS is used for **persistent configuration only** (no keys, no PBFT state that requires durability beyond a single boot — see CRYPTO.md §11).

### 4.1 Namespaces and keys

```
Namespace: "pbft"
  Key                Type    Size    Default    Purpose
  ──────────────────────────────────────────────────────────
  "my_node_id"       u8      1       0          this node's ID
  "view"             u32     4       0          last view (recovery)
  "next_seq"         u64     8       0          last next_sequence
  "low_wm"           u64     8       0          last low_watermark
  "high_wm"          u64     8       0          last high_watermark
  "log_size_max"     u16     2       100        PBFT_LOG_MAX_ENTRIES
  "payload_max"      u16     2       256        PBFT_TX_PAYLOAD_MAX
  "transport"        u8      1       0          0=ESP-NOW, 1=Wi-Fi UDP

Namespace: "pbft_cluster"
  Key                Type    Size    Purpose
  ──────────────────────────────────────────────────────────────────────────
  "node_<id>_mac"    blob    6       MAC address for node <id>
  "node_<id>_ip"     u32     4       IPv4 (host-order) for node <id>
  "cluster_size"     u8      1       must be 7

Namespace: "pbft_timeouts"
  Key                Type    Size    Default    Purpose
  ──────────────────────────────────────────────────────────
  "prep_to"          u32     4       1000       prepare_timeout_ms
  "commit_to"        u32     4       1000       commit_timeout_ms
  "vc_to"            u32     4       2000       viewchange_timeout_ms
  "ckpt_to"          u32     4       2000       checkpoint_timeout_ms
```

Total NVS footprint: ~120 B per namespace (ESP-IDF overhead) + ~80 B data = **~280 B across 3 namespaces**.

### 4.2 Read/write policy

- All keys written once at provisioning (factory), except `view`/`next_seq`/`low_wm`/`high_wm` which are written on graceful shutdown (`pbft_stop`).
- On unclean shutdown (power loss), the values from the last clean shutdown are used. The cluster will re-converge via normal view-change + New-View catch-up.
- NVS reads happen at `pbft_init`; writes happen at `pbft_stop` and at view-change boundaries (optional, for fast recovery).

---

## 5. Heap usage (libraries, not esp-pbft source)

### 5.1 PSA crypto internal allocations

PSA `psa_mac_compute` allocates a transient internal buffer (typically 256-1024 B) per call. With ESP-IDF's `CONFIG_MBEDTLS_DYNAMIC_BUFFER=y` (default), this is freed after each call.

| Call | Transient heap peak | Frequency |
|------|--------------------|-----------|
| `psa_generate_key` | ~1 KB | once per boot + every Y-5 |
| `psa_key_agreement` | ~1 KB | 6 × per boot + per peer re-handshake |
| `psa_mac_compute` | ~512 B | per message sent |
| `psa_mac_verify` | ~512 B | per message received |

**Steady-state heap pressure:** ~1 KB transient, freed within microseconds.

### 5.2 FreeRTOS

- Tasks: 7 × 2 KB stack each (main, PBFT, idle, ESP-NOW RX, ESP-NOW TX, event loop, sys) = ~14 KB
- Queues: TX queue 4.5 KB already counted in §2.13; RX queue ~2 KB
- Mutexes: ~100 B each × 4 = ~400 B

### 5.3 ESP-NOW internals

ESP-NOW uses ~5 KB of internal heap for peer table + encryption scratch. Allocated once at `esp_now_init`, freed at `esp_now_deinit`.

### 5.4 Total heap peak

| Event | Heap peak |
|-------|-----------|
| Boot (TRNG keygen + ECDH × 6) | ~6 KB transient |
| Y-5 re-gen | ~3 KB transient |
| Steady-state (per message) | ~1 KB transient |
| Wi-Fi UDP steady-state | ~2 KB transient |

**ESP32-C3 heap at boot:** ~250 KB free (after ESP-IDF init). Worst transient ~6 KB → **2.4% peak heap pressure**.

### 5.5 Heap tracking

`pbft_metrics_t.heap_free_bytes` and `heap_min_free_bytes` are sampled every 60 s and on every state transition. If `heap_min_free_bytes < 16 KB`, log alarm.

---

## 6. Stack analysis

### 6.1 Per-task stack

| Task | Stack | Notes |
|------|-------|-------|
| `pbft_task` | 8 KB | handles consensus, RX, timers |
| `main` (app) | 8 KB | application code |
| `esp_event` | 4 KB | ESP-IDF event loop |
| ESP-NOW RX callback | 2 KB | runs in Wi-Fi task context |
| `esp_timer` | 2 KB | timer service |

### 6.2 Deep call chains

The deepest consensus call chain:
```
pbft_handle_new_view (frame: 200 B)
  → pbft_verify_nv_outer_mac (frame: 80 B)
  → pbft_compute_o_set (frame: ~100 B + O-set array)
  → pbft_handle_pre_prepare (frame: 200 B)
    → pbft_send_prepare (frame: 100 B)
      → pbft_compute_mac (frame: 45 + payload + 32 B)
        → psa_mac_compute (PSA internal: ~500 B)
```

**Worst-case stack depth:** ~200 + 80 + 100 + 200 + 100 + 100 + 500 = **~1280 B**.

`pbft_task` stack of 8 KB is comfortable.

### 6.3 Stack canary

ESP-IDF's `CONFIG_COMPILER_STACK_CHECK_MODE_STRONG=y` adds `-fstack-protector-strong` which inserts canaries per function. esp-pbft enables this in its component CMakeLists:

```cmake
target_compile_options(${COMPONENT_LIB} PRIVATE
    -fstack-protector-strong
)
```

---

## 7. Fragmentation safety

### 7.1 Why static allocation

| Benefit | Impact |
|---------|--------|
| Deterministic boot | Memory layout is identical across boots → easier to debug |
| No fragmentation | Long-running clusters (months/years) don't degrade |
| Predictable budget | Compile-time verification that total ≤ budget |
| No ENOMEM | Hot path never blocks on allocation |
| OTA-friendly | Memory layout stable across firmware versions |

### 7.2 Allocation pattern audit

```bash
# Verify zero heap allocations in esp-pbft source:
git grep -nE '\b(malloc|calloc|realloc|free|alloca|_alloca)\b' src/ include/
# Expected: zero matches (except in this MEMORY.md explaining the discipline)
```

CI gate: this grep must return zero matches.

### 7.3 Heap fragmentation mitigation

For PSA / FreeRTOS / ESP-NOW heap usage:
- `CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE=4096` (default; don't change)
- `CONFIG_MBEDTLS_DYNAMIC_BUFFER=y` (default; don't change)
- Avoid fragmentation-inducing patterns: no large transient allocations, no many small allocations

### 7.4 Heap fragmentation detector

esp-pbft optionally enables heap poisoning in test builds:

```c
#ifdef CONFIG_PBFT_HEAP_POISONING
void* pbft_test_alloc(size_t size) {
    void* p = malloc(size);
    if (p) memset(p, 0xA5, size);  // poison
    return p;
}
void pbft_test_free(void* p) {
    // check for use-after-free
    free(p);
}
#endif
```

This catches buffer overflows in tests but is **disabled** in production (`CONFIG_PBFT_HEAP_POISONING=n`).

---

## 8. Compile-time bounds checking (full catalog)

The complete `_Static_assert` catalog that lives in `pbft_config.h`:

```c
// === Cluster ===
_Static_assert(PBFT_CLUSTER_SIZE == 7,                "PBFT_CLUSTER_SIZE must be 7");
_Static_assert(PBFT_F == 2,                            "PBFT_F must be 2");
_Static_assert(PBFT_QUORUM == (2 * PBFT_F + 1),       "PBFT_QUORUM must equal 2f+1");

// === Log ===
_Static_assert(PBFT_LOG_MAX_ENTRIES >= PBFT_CHECKPOINT_INTERVAL,
               "LOG_MAX_ENTRIES must be >= CHECKPOINT_INTERVAL");
_Static_assert(PBFT_LOG_MAX_ENTRIES >= 50,             "LOG_MAX_ENTRIES too small");
_Static_assert(PBFT_LOG_MAX_ENTRIES <= 1000,           "LOG_MAX_ENTRIES too large for SRAM");
_Static_assert(sizeof(pbft_log_entry_t) <= 384,
               "pbft_log_entry_t exceeded 384 B sub-budget");
_Static_assert(sizeof(pbft_log_entry_t) * PBFT_LOG_MAX_ENTRIES <= 40 * 1024,
               "PBFT log exceeded 40 KB total budget");

// === TX payload ===
_Static_assert(PBFT_TX_PAYLOAD_MAX >= 1,              "TX_PAYLOAD_MAX must be >= 1");
_Static_assert(PBFT_TX_PAYLOAD_MAX <= 256,             "TX_PAYLOAD_MAX must be <= 256");

// === Crypto ===
_Static_assert(PBFT_HMAC_KEY_BYTES == 32,              "HMAC-SHA256 key size must be 32");
_Static_assert(PBFT_DIGEST_BYTES == 32,                "SHA-256 digest size must be 32");
_Static_assert(PBFT_PUBKEY_BYTES == 65,                "P-256 uncompressed pubkey must be 65 B (0x04 ‖ X_BE ‖ Y_BE)");

// === Transport (compile-time only — no runtime fallback) ===
#ifdef CONFIG_PBFT_TRANSPORT_ESP_NOW
// If ESP-NOW is selected, Pre-Prepare with max payload MUST fit in 250 B.
// No runtime fallback to UDP exists (user requirement). Either reduce
// PBFT_TX_PAYLOAD_MAX, or switch to CONFIG_PBFT_TRANSPORT_WIFI_UDP=y.
// Note: View-Change can also exceed 250 B (up to 4 KB worst case at
// PBFT_VC_MAX_PREPARED=100); if ESP-NOW is selected with that config,
// build will fail here.
_Static_assert(PBFT_TX_PAYLOAD_MAX + 82 <= 250,
               "Pre-Prepare with max payload exceeds ESP-NOW 250 B limit — "
               "either reduce PBFT_TX_PAYLOAD_MAX or select "
               "CONFIG_PBFT_TRANSPORT_WIFI_UDP=y (compile-time only, no runtime fallback)");
_Static_assert(sizeof(pbft_view_change_t) <= 250,
               "View-Change worst case exceeds ESP-NOW 250 B — "
               "either reduce PBFT_VC_MAX_PREPARED or select "
               "CONFIG_PBFT_TRANSPORT_WIFI_UDP=y (compile-time only, no runtime fallback)");
// New-View (~5.9 KB worst case) NEVER fits in ESP-NOW. If ESP-NOW is
// selected, view-change is impossible and the cluster is stuck on view 0.
// This is by design (user requirement: no runtime fallback). To enable
// view-change, select CONFIG_PBFT_TRANSPORT_WIFI_UDP=y.
#endif

// === Memory totals ===
_Static_assert(sizeof(pbft_log_entry_t) * PBFT_LOG_MAX_ENTRIES +
               sizeof(pbft_view_change_t) * 7 +
               sizeof(pbft_checkpoint_proof_t) * 8 +
               72 * 1024   // misc + crypto + dedup + queues
               <= 150 * 1024,
               "esp-pbft static memory exceeded 150 KB");
```

The last assertion is the most important — it fails the build if anyone adds a new buffer that pushes the total over the budget.

---

## 9. Memory audit checklist (CI gate)

Every PR must include in the description:

- [ ] No `malloc`/`calloc`/`realloc`/`free` in new code (grep verification)
- [ ] No `xQueueCreate`/`xTaskCreate` with dynamic sizes
- [ ] All new buffers declared `static` or as BSS arrays
- [ ] New `_Static_assert` added for any new compile-time bounds
- [ ] Inventory tables in this document updated
- [ ] `idf.py size-components` output attached (showing no unexpected growth)
- [ ] Heap peak measurement: `heap_min_free_bytes ≥ 16 KB` after 1-hour soak test

---

## 10. Open questions (memory level)

| # | Question | Recommendation |
|---|----------|----------------|
| M1 | Can we reduce `PBFT_TX_PAYLOAD_MAX` to 128 B (cut log from 36 KB to 24 KB)? | Yes if app doesn't need 256 B. Configurable. |
| M2 | Should we compress checkpoint proof table (8 → 4)? | No — 640 B is not worth the complexity. |
| M3 | Should dedup cache be smaller (256 → 64 entries = 2.5 KB saved)? | Yes for memory-tight builds; default keep 256. |
| M4 | Should we exclude lwIP from ESP-NOW build? | Yes — saves 30 KB. Add `CONFIG_LWIP_ENABLE=n` to default config. |

---

## 11. References

- [HANDOVER.md](./HANDOVER.md) §3.7 (now superseded — original "40 KB total" claim was inaccurate)
- [ARCHITECTURE.md](./ARCHITECTURE.md) §11 (partial inventory; now completed here)
- [CHECKPOINT.md](./CHECKPOINT.md) §10 — checkpoint-related memory
- [VIEW-CHANGE.md](./VIEW-CHANGE.md) §4.1 — vc_set_arena
- [API-REFERENCE.md](./API-REFERENCE.md) §6 — memory contract
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — A5 (per-entry size), B5 (bitmask), B6 (state_entered_ms), B10 (dedup cache)

---

**End of MEMORY.md (v0.1 draft — addresses audit issue A5 and corrects memory budget)**