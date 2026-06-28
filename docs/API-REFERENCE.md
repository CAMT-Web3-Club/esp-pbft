# esp-pbft — Public API Reference

> **Status:** 🟡 Draft (v0.1)
> **Scope:** complete public API contract — signatures, preconditions, postconditions, error codes, thread-safety, ownership/lifetime, callback contracts, Kconfig switches.
> **Audience:** application developers integrating esp-pbft into their firmware.
> **Out of scope:** internal (non-public) APIs; implementation details.

This document is the **canonical reference for the esp-pbft public C API**. It defines everything an application needs to call esp-pbft and every contract esp-pbft will uphold. It addresses audit issues **C3** (`pbft_metrics_t` undefined), **C4** (error codes not enumerated), and **C5** (`pbft_config_t` undefined).

---

## 1. Header and version

```c
#include "pbft.h"
```

The header is the only file applications need to include. Internal headers (`pbft_consensus.h`, `pbft_crypto.h`, etc.) are **not** installed and must not be included from application code.

```c
#define PBFT_VERSION_MAJOR  0
#define PBFT_VERSION_MINOR  1
#define PBFT_VERSION_PATCH  0
#define PBFT_VERSION_STRING "0.1.0"
```

---

## 2. Configuration

### 2.1 `pbft_config_t`

```c
typedef struct {
    uint8_t  my_node_id;                     // 0..6 (PBFT_CLUSTER_SIZE-1)
    uint8_t  cluster_size;                   // must be 7 (compile-time constant)
    const pbft_node_desc_t* cluster_members; // array of length cluster_size
    uint32_t prepare_timeout_ms;             // default 1000
    uint32_t commit_timeout_ms;              // default 1000
    uint32_t viewchange_timeout_ms;          // default 2000
    uint8_t  payload_max;                    // 1..256 (PBFT_TX_PAYLOAD_MAX)
    pbft_state_digest_fn state_digest_cb;    // optional; NULL = all-zero digest
    pbft_log_fn           log_cb;           // optional; NULL = silent
} pbft_config_t;
```

### 2.2 `pbft_node_desc_t`

```c
typedef struct {
    uint8_t  node_id;            // 0..6
    uint8_t  mac_addr[6];        // ESP-NOW MAC (FF:FF:FF:FF:FF:FF for broadcast slot)
    uint32_t ipv4_addr;          // Wi-Fi UDP IP, host-order (0 if ESP-NOW only)
} pbft_node_desc_t;
```

### 2.3 Lifecycle

```c
esp_err_t pbft_init(const pbft_config_t* config);
esp_err_t pbft_start(void);
esp_err_t pbft_stop(void);
esp_err_t pbft_deinit(void);
```

| Function | Contract |
|----------|----------|
| `pbft_init` | Validates `config`. Generates TRNG keypair. Stores config in BSS. Returns `ESP_OK` or one of [§3 error codes]. Idempotent (subsequent calls return `PBFT_ERR_ALREADY_INITIALIZED`). |
| `pbft_start` | Begins Pattern Y handshake (Hello broadcast + ECDH with peers). Blocks for up to 5 s for Hello exchange. Returns `ESP_OK` once at least `PBFT_QUORUM - 1 = 4` peer Hellos received, OR `PBFT_ERR_HANDSHAKE_TIMEOUT` after 5 s. After timeout, retries once; persistent failure returns `PBFT_ERR_NO_PEERS`. |
| `pbft_stop` | Graceful shutdown: cancels timers, deregisters ESP-NOW callbacks, does NOT destroy keys (so re-start doesn't trigger re-handshake). Idempotent. |
| `pbft_deinit` | Destroys keys, zeros RAM, releases ESP-NOW peers. Idempotent. After `pbft_deinit`, must call `pbft_init` again before `pbft_start`. |

---

## 3. Error codes

```c
typedef enum {
    // Success
    PBFT_OK                     = 0,

    // Generic
    PBFT_ERR_INVALID_ARG        = -1,   // NULL pointer, out-of-range arg
    PBFT_ERR_INVALID_STATE      = -2,   // operation not valid in current state
    PBFT_ERR_NO_MEM             = -3,   // (reserved — should never fire; see §6)
    PBFT_ERR_TIMEOUT            = -4,   // operation timed out
    PBFT_ERR_BUSY               = -5,   // concurrent operation in progress

    // Lifecycle
    PBFT_ERR_ALREADY_INITIALIZED = -10,
    PBFT_ERR_NOT_INITIALIZED     = -11,
    PBFT_ERR_ALREADY_STARTED     = -12,
    PBFT_ERR_NOT_STARTED         = -13,

    // Handshake / discovery
    PBFT_ERR_NO_PEERS            = -20,  // < 4 peer Hellos within timeout
    PBFT_ERR_HANDSHAKE_TIMEOUT   = -21,  // timed out mid-handshake
    PBFT_ERR_INVALID_PEER        = -22,  // TOFU pubkey mismatch

    // Crypto
    PBFT_ERR_INVALID_MAC         = -30,  // HMAC verify failed
    PBFT_ERR_CRYPTO_INIT         = -31,  // PSA init failed
    PBFT_ERR_KEYGEN              = -32,  // psa_generate_key failed
    PBFT_ERR_KEY_AGREE           = -33,  // psa_key_agreement failed

    // Consensus
    PBFT_ERR_NOT_PRIMARY         = -40,  // submit called on non-primary
    PBFT_ERR_LOG_FULL            = -41,  // high_watermark reached
    PBFT_ERR_SEQ_OUT_OF_RANGE    = -42,  // Pre-Prepare seq outside [low+1, high]
    PBFT_ERR_VIEW_MISMATCH       = -43,  // msg view != current view
    PBFT_ERR_PRIMARY_MISMATCH    = -44,  // msg sender != current primary

    // Network
    PBFT_NET_ERR_TIMEOUT         = -50,  // transport timeout
    PBFT_NET_ERR_INVALID         = -51,  // bad packet
    PBFT_NET_ERR_FULL            = -52,  // TX buffer full
    PBFT_NET_ERR_NO_ROUTE        = -53,  // peer unknown

    // Internal (not propagated to app; logged only)
    PBFT_ERR_INTERNAL            = -100,
} pbft_err_t;
```

`esp_err_t` values returned to the app are guaranteed to be in the range `[PBFT_OK, PBFT_NET_ERR_NO_ROUTE]`; `PBFT_ERR_INTERNAL` and below are never returned (logged instead).

> **Note — transport-status enum is separate.** The `PBFT_NET_ERR_*` codes above (`-50..-53`) belong to `pbft_err_t`. The transport layer uses a **distinct** enum, `pbft_transport_status_t`, defined in [PROTOCOL.md §2.1](./PROTOCOL.md) with values `PBFT_TX_OK = 0`, `PBFT_TX_TIMEOUT = -1`, `PBFT_TX_INVALID = -2`, `PBFT_TX_FULL = -3`. The transport layer **maps** `pbft_transport_status_t` into `pbft_err_t` (e.g. `PBFT_TX_TIMEOUT` → `PBFT_NET_ERR_TIMEOUT`). The two enums are not the same symbols and must not be confused.

---

## 4. Transaction submission

### 4.1 `pbft_tx_t`

```c
typedef struct {
    const uint8_t* payload;       // 0..PBFT_TX_PAYLOAD_MAX bytes
    size_t         payload_len;   // 0..PBFT_TX_PAYLOAD_MAX
    uint8_t        digest[32];    // OUT only — esp-pbft fills this in
    uint64_t       sequence;      // OUT only — assigned by primary
    uint32_t       flags;         // IN — see below
} pbft_tx_t;

// Flag bits:
#define PBFT_TX_FLAG_NONE      0x00
#define PBFT_TX_FLAG_DEDUP     0x01   // reject duplicate digest (audit B10)
#define PBFT_TX_FLAG_NOOP      0x02   // no-op (digest = zeros, payload ignored)
```

### 4.2 `pbft_submit`

```c
esp_err_t pbft_submit(pbft_tx_t* tx);   // NON-const: tx->digest and tx->sequence are written back
```

`tx` is **non-const** because `pbft_submit` writes back two OUT fields on success:

- `tx->digest[32]` — the SHA-256 of the request payload (computed via `pbft_sha256(...)`, see CRYPTO.md).
- `tx->sequence` — the sequence number assigned by the primary. If the digest was already in the dedup cache (§15.4), this is the **existing** sequence rather than a newly assigned one.

| Aspect | Contract |
|--------|----------|
| **Preconditions** | `pbft_started` is true; `tx != NULL`; `tx->payload != NULL || tx->payload_len == 0`; `tx->payload_len <= PBFT_TX_PAYLOAD_MAX`. |
| **Caller must be primary** | If `my_node_id != view_state.primary_id`, returns `PBFT_ERR_NOT_PRIMARY`. App is responsible for tracking the current primary via `pbft_get_view()` / `pbft_get_state()`. |
| **OUT fields (written on success)** | `tx->digest` (SHA-256 of payload) and `tx->sequence` (assigned/existing sequence). Undefined on error. |
| **Side effects** | Computes digest (fills `tx->digest`); assigns sequence (fills `tx->sequence`); broadcasts Pre-Prepare. |
| **Return value** | `ESP_OK` when Pre-Prepare was successfully enqueued for transmission. **Not** when the cluster commits — that is asynchronous and signaled via the commit callback. |
| **Errors** | `PBFT_ERR_NOT_PRIMARY`, `PBFT_ERR_LOG_FULL`, `PBFT_ERR_INVALID_ARG`. |
| **Thread-safety** | Must be called from the PBFT task (or with the FreeRTOS lock if `CONFIG_PBFT_MULTI_TASK_API=y`). See §8. |

### 4.3 Example

```c
void app_submit_counter_increment(int delta) {
    if (pbft_get_state() != PBFT_STATE_NORMAL) {
        ESP_LOGW(TAG, "Cluster not ready (state=%d)", pbft_get_state());
        return;
    }
    if (pbft_get_primary_id() != MY_EXPECTED_PRIMARY) {
        // Forwarding is the application's responsibility (audit A3)
        // For multi-primary scenarios, see §10
        return;
    }
    pbft_tx_t tx = {
        .payload = (uint8_t*)&delta,
        .payload_len = sizeof(delta),
        .flags = PBFT_TX_FLAG_DEDUP,
    };
    esp_err_t err = pbft_submit(&tx);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "pbft_submit failed: %s", esp_err_to_name(err));
    }
}
```

---

## 5. Commit callback

### 5.1 `pbft_commit_cb_t`

```c
typedef void (*pbft_commit_cb_t)(const pbft_tx_t* tx, uint64_t sequence, void* user_ctx);
```

### 5.2 Registration

```c
esp_err_t pbft_register_commit_cb(pbft_commit_cb_t cb, void* user_ctx);
```

| Aspect | Contract |
|--------|----------|
| **Preconditions** | `pbft_started` is true; `cb != NULL`. |
| **Side effects** | Stores the callback in BSS. Only ONE callback may be registered at a time; re-registering replaces the previous one. |
| **Invocation timing** | Called from the PBFT task when the local replica reaches the COMMITTED state for a TX. **Not** when the entire cluster commits — that's a stronger guarantee that requires the application to wait for `n` callbacks (rare; PBFT safety ensures all honest replicas commit the same TX). |
| **Re-entrancy** | The callback may call any esp-pbft API EXCEPT `pbft_stop`, `pbft_deinit`, `pbft_init`, `pbft_register_commit_cb`. |
| **Blocking** | Must return promptly (≤ 1 ms recommended). Long-running work must be deferred to an app task. |
| **Payload lifetime** | `tx->payload` points to memory inside the PBFT log; valid for the duration of the callback ONLY. Copy if needed. |
| **Thread-safety** | Called from the PBFT task; app callback runs in that context. |

### 5.3 Example

```c
static void on_commit(const pbft_tx_t* tx, uint64_t seq, void* ctx) {
    int delta;
    memcpy(&delta, tx->payload, sizeof(delta));
    g_counter += delta;
    ESP_LOGI(TAG, "seq=%llu delta=%d counter=%d", seq, delta, g_counter);
}

void app_start(void) {
    pbft_register_commit_cb(on_commit, NULL);
    pbft_start();
}
```

---

## 6. Memory contract

esp-pbft allocates **zero bytes from the heap** in its source code. All internal state lives in BSS:

- PBFT log: ~36 KB
- Crypto state: ~3 KB
- View state + V-set: ~5 KB
- Network buffers: ~8 KB
- Misc (metrics, config): ~1 KB
- **Total: ~53 KB** (ESP-NOW build), **~83 KB** (Wi-Fi UDP build, +30 KB for UDP stack)

Heap usage from PSA crypto, FreeRTOS, and ESP-NOW internals is bounded by Kconfig (e.g., `CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE`) and reproducible.

`PBFT_ERR_NO_MEM` is **reserved** — it must never be returned. If it ever fires, it's a bug.

### 6.1 Static_assert catalog

```c
// In pbft_config.h — verified at compile time

_Static_assert(PBFT_CLUSTER_SIZE == 7, "PBFT_CLUSTER_SIZE must be 7");
_Static_assert(PBFT_F == 2, "PBFT_F must be 2");
_Static_assert(PBFT_QUORUM == 5, "PBFT_QUORUM must be 5");
_Static_assert(PBFT_LOG_MAX_ENTRIES >= PBFT_CHECKPOINT_INTERVAL,
               "LOG_MAX_ENTRIES must be >= CHECKPOINT_INTERVAL");
_Static_assert(PBFT_TX_PAYLOAD_MAX <= 256, "TX_PAYLOAD_MAX must be <= 256");
_Static_assert(sizeof(pbft_log_entry_t) * PBFT_LOG_MAX_ENTRIES <= 40 * 1024,
               "PBFT log exceeds 40 KB budget");
```

---

## 7. State inspection

### 7.1 State machine (from the outside)

```c
typedef enum {
    PBFT_STATE_UNINIT      = 0,   // before pbft_init
    PBFT_STATE_INIT        = 1,   // after pbft_init, before pbft_start
    PBFT_STATE_HANDSHAKE   = 2,   // pbft_start in progress
    PBFT_STATE_NORMAL      = 3,   // steady state — accepting submissions
    PBFT_STATE_VIEW_CHANGE = 4,   // mid view-change
    PBFT_STATE_PARTITION   = 5,   // declared partition mode (no progress)
    PBFT_STATE_STOPPED     = 6,   // after pbft_stop
} pbft_state_t;

pbft_state_t pbft_get_state(void);
uint32_t     pbft_get_view(void);
uint8_t      pbft_get_primary_id(void);
uint64_t     pbft_get_sequence(void);   // next-to-assign
uint64_t     pbft_get_last_committed(void);
uint64_t     pbft_get_last_executed(void);
uint64_t     pbft_get_stable_checkpoint(void);
```

| Function | Returns |
|----------|---------|
| `pbft_get_state` | Current state (see enum). |
| `pbft_get_view` | Current view number (0 at boot, increments per view-change). |
| `pbft_get_primary_id` | `view % PBFT_CLUSTER_SIZE`. |
| `pbft_get_sequence` | Primary's next-to-assign sequence number. |
| `pbft_get_last_committed` | Highest locally-committed sequence. |
| `pbft_get_last_executed` | Highest locally-executed sequence (= last callback fired). |
| `pbft_get_stable_checkpoint` | Highest stable checkpoint sequence. |

All getters are **thread-safe** (atomic reads from BSS; called from any task).

### 7.2 Per-entry state vs. node state — two DISTINCT enums

`pbft_state_t` (above, prefix `PBFT_STATE_*`) is the **node-level** state machine returned by `pbft_get_state()`. It is **not** the same as the **per-log-entry** state used internally by the consensus log. Each log entry carries a `pbft_entry_state_t` (prefix `PBFT_ENTRY_*`), defined in MEMORY.md §2.3 / CONSENSUS.md §3.1:

```c
typedef enum {
    PBFT_ENTRY_FREE = 0,        // empty slot (NOT "IDLE")
    PBFT_ENTRY_PRE_PREPARED,
    PBFT_ENTRY_PREPARED,
    PBFT_ENTRY_COMMITTED,
    PBFT_ENTRY_EXECUTED,
} pbft_entry_state_t;
```

The `PBFT_STATE_*` prefix must never be reused for log-entry states, and `pbft_entry_state_t` is internal (not part of the public app API). See [MEMORY.md §2.3](./MEMORY.md) for the canonical `pbft_log_entry_t` that embeds this field.

---

## 8. Thread-safety contract

### 8.1 Default: single-task API (`CONFIG_PBFT_MULTI_TASK_API=n`)

All API calls (except getters) must originate from the PBFT task. The PBFT task is created internally by `pbft_start`; its priority is `tskIDLE_PRIORITY + 5` (configurable via `CONFIG_PBFT_TASK_PRIORITY`).

To call `pbft_submit` from another task, use the queue-based variant:

```c
esp_err_t pbft_submit_from_isr(pbft_tx_t* tx);  // callable from ISR or other task
```

This copies the TX into the PBFT TX queue and returns immediately. The PBFT task drains the queue and runs the consensus algorithm.

### 8.2 Multi-task API (`CONFIG_PBFT_MULTI_TASK_API=y`)

All public API calls are protected by an internal mutex (`pbft_api_mutex`, FreeRTOS recursive). Slight overhead per call (~1 µs) but allows direct calls from any task. `pbft_submit_from_isr` becomes a thin wrapper around `xQueueSendFromISR`.

---

## 9. Metrics

### 9.1 `pbft_metrics_t`

```c
typedef struct {
    uint64_t tx_submitted;          // total pbft_submit calls
    uint64_t tx_committed_local;    // commit callbacks fired
    uint64_t tx_committed_cluster;  // TXs where local + 2f+1 peers all committed (estimated)
    uint64_t view_changes_initiated;
    uint64_t view_changes_completed;
    uint64_t checkpoints_stable;
    uint64_t prepare_timeouts;
    uint64_t commit_timeouts;
    uint64_t mac_failures;          // HMAC verify failed (potential attack indicator)
    uint64_t retransmits;
    uint32_t current_view;
    uint64_t current_sequence;
    uint64_t current_low_watermark;
    uint64_t current_high_watermark;
    uint32_t heap_free_bytes;       // ESP_GET_FREE_HEAP_SIZE at sample time
    uint32_t heap_min_free_bytes;   // since boot
} pbft_metrics_t;

pbft_metrics_t pbft_get_metrics(void);
void           pbft_reset_metrics(void);
```

`pbft_get_metrics` returns a snapshot (atomic read of BSS struct). `pbft_reset_metrics` zeros the counters; thread-safe.

---

## 10. Multi-primary forwarding (v2 / workaround for audit A3)

For applications that cannot tolerate `PBFT_ERR_NOT_PRIMARY`, esp-pbft provides a forwarding helper:

```c
typedef enum {
    PBFT_FWD_DIRECT   = 0,   // I am primary — submit locally
    PBFT_FWD_UNICAST  = 1,   // forward to current primary via UDP unicast
    PBFT_FWD_BROADCAST= 2,   // broadcast (let primary pick it up)
} pbft_forward_mode_t;

esp_err_t pbft_submit_with_forward(pbft_tx_t* tx, pbft_forward_mode_t mode);
```

- `PBFT_FWD_DIRECT`: behaves like `pbft_submit` on primary; returns `PBFT_ERR_NOT_PRIMARY` on replica.
- `PBFT_FWD_UNICAST`: on a replica, sends the TX to the current primary via UDP unicast. The primary then runs `pbft_submit` locally.
- `PBFT_FWD_BROADCAST`: on a replica, broadcasts the TX; the primary picks it up and runs `pbft_submit`. The original sender also runs `pbft_submit` on its own node to track the TX (and gets a local commit callback).

`PBFT_FWD_BROADCAST` is the recommended default for fault-tolerant submissions. Cost: ~10 ms extra latency vs direct submission.

**v1 limitation:** forwarding is best-effort. If the primary changes between `pbft_get_primary_id()` and the actual forwarding, the TX may be lost. The application should track `view` and retry on view-change.

---

## 11. Configuration constants (Kconfig summary)

```kconfig
# Cluster
CONFIG_PBFT_CLUSTER_SIZE=7
CONFIG_PBFT_F=2

# Timeouts
CONFIG_PBFT_PREPARE_TIMEOUT_MS=1000
CONFIG_PBFT_COMMIT_TIMEOUT_MS=1000
CONFIG_PBFT_VIEWCHANGE_TIMEOUT_MS=2000
CONFIG_PBFT_CHECKPOINT_TIMEOUT_MS=2000

# Intervals
CONFIG_PBFT_LOG_MAX_ENTRIES=100
CONFIG_PBFT_CHECKPOINT_INTERVAL=100
CONFIG_PBFT_TX_PAYLOAD_MAX=256

# Transport (mutually exclusive)
CONFIG_PBFT_TRANSPORT_ESP_NOW=y
# CONFIG_PBFT_TRANSPORT_WIFI_UDP is not set

# Wi-Fi UDP power-save mode (only effective when WIFI_UDP=y)
# See POWER.md §12 for trade-offs.
CONFIG_PBFT_WIFI_PS_MIN_MODEM=y
# CONFIG_PBFT_WIFI_PS_NONE=y       # debug only
# CONFIG_PBFT_WIFI_PS_MAX_MODEM=y  # marginal extra savings

# Crypto
CONFIG_PBFT_Y5_REGEN_PERIOD_HOURS=24
CONFIG_PBFT_Y5_STAGGER_HOURS=1
CONFIG_PBFT_BOOT_STAGGER_MS=3000

# Concurrency
CONFIG_PBFT_MULTI_TASK_API=n
CONFIG_PBFT_TASK_PRIORITY=5
CONFIG_PBFT_TASK_STACK_SIZE=8192

# State transfer
CONFIG_PBFT_STATE_TRANSFER=y
```

---

## 12. Example: minimal counter application

```c
// app_main.c
#include <stdio.h>
#include "pbft.h"

static int g_counter = 0;
static const char* TAG = "app";

static void on_commit(const pbft_tx_t* tx, uint64_t seq, void* ctx) {
    int delta;
    if (tx->payload_len != sizeof(delta)) return;
    memcpy(&delta, tx->payload, sizeof(delta));
    g_counter += delta;
    ESP_LOGI(TAG, "seq=%llu delta=%d counter=%d", seq, delta, g_counter);
}

void app_main(void) {
    // 1. Cluster configuration (baked at provisioning)
    static const pbft_node_desc_t cluster[7] = {
        {.node_id = 0, .mac_addr = {0xAA,0xBB,0xCC,0xDD,0xEE,0x00}, .ipv4_addr = 0},
        // ... 6 more ...
    };

    pbft_config_t cfg = {
        .my_node_id          = 0,    // set per-device at provisioning
        .cluster_size        = 7,
        .cluster_members     = cluster,
        .prepare_timeout_ms  = 1000,
        .commit_timeout_ms   = 1000,
        .viewchange_timeout_ms = 2000,
        .payload_max         = 256,
        .state_digest_cb     = NULL,
        .log_cb              = NULL,
    };

    ESP_ERROR_CHECK(pbft_init(&cfg));
    ESP_ERROR_CHECK(pbft_register_commit_cb(on_commit, NULL));
    ESP_ERROR_CHECK(pbft_start());

    // 2. Submit TXs from app logic
    int delta = 1;
    pbft_tx_t tx = {
        .payload = (uint8_t*)&delta,
        .payload_len = sizeof(delta),
        .flags = PBFT_TX_FLAG_DEDUP,
    };
    pbft_submit(&tx);
}
```

Full example in [DEPLOYMENT.md §5](./DEPLOYMENT.md).

---

## 13. Versioning and stability

- **Source compatibility:** minor versions (0.x) may add APIs and break source compatibility. Major versions (1.0, 2.0) maintain source compatibility within the major.
- **Wire compatibility:** minor versions MUST NOT break the wire format. Wire-format changes require a major version bump.
- **ABI:** no ABI stability guarantee until v1.0.

---

## 14. Open questions (API level)

| # | Question | Recommendation |
|---|----------|----------------|
| API1 | Should `pbft_submit` block until committed? | No — fire-and-forget is the PBFT model. Async via callback. |
| API2 | Should we expose internal state (e.g., per-entry bitmask)? | No — internal API. Use `pbft_get_metrics`. |
| API3 | Should `pbft_register_commit_cb` allow multiple callbacks? | v1: single callback. v2: array. |
| API4 | Should we expose `pbft_y5_force_regen()` for testing? | Yes — add as `pbft_y5_force_regen()` debug API, gated by `CONFIG_PBFT_DEBUG=y`. |

---

## 15. Internal API (used across modules, not for applications)

These functions are referenced throughout CONSENSUS.md, VIEW-CHANGE.md, and CHECKPOINT.md but are NOT part of the public application API. They are declared in internal headers (`pbft_consensus_internal.h`, `pbft_log.h`, etc.).

### 15.1 Log entry management (CONSENSUS.md §3.1)

```c
// pbft_log.h
pbft_log_entry_t* pbft_log_get_or_create(uint32_t view, uint64_t sequence);
pbft_log_entry_t* pbft_log_find(uint32_t view, uint64_t sequence);
void              pbft_log_free(pbft_log_entry_t* entry);
bool              pbft_log_full(void);
```

| Function | Purpose | Used by |
|----------|---------|---------|
| `pbft_log_get_or_create` | Find existing entry or allocate new slot for (view, seq). Returns NULL if log is full. | CONSENSUS.md §4.2 (Pre-Prepare handler) |
| `pbft_log_find` | Find existing entry for (view, seq). Returns NULL if absent. | CONSENSUS.md §4.3-4.4 (Prepare/Commit handlers) |
| `pbft_log_free` | Mark entry as FREE and zeroize its payload. | CHECKPOINT.md §7.2 (GC) |
| `pbft_log_full` | True when `next_sequence > high_watermark`. | CONSENSUS.md §5.2 (submit gate) |

**Threading:** called only from the PBFT task (single-threaded).

### 15.2 Send primitives (CONSENSUS.md §4, CHECKPOINT.md §5)

```c
// pbft_send.h
esp_err_t pbft_send_pre_prepare(const pbft_pre_prepare_t* pp);
esp_err_t pbft_send_prepare(const pbft_prepare_t* p);
esp_err_t pbft_send_commit(const pbft_commit_t* c);
esp_err_t pbft_resend_prepare(const pbft_log_entry_t* entry);  // CONSENSUS §6.3
esp_err_t pbft_resend_commit(const pbft_log_entry_t* entry);   // CONSENSUS §6.3
esp_err_t pbft_send_view_change(const pbft_view_change_t* vc);
esp_err_t pbft_send_new_view(const pbft_new_view_t* nv);
esp_err_t pbft_send_checkpoint(const pbft_checkpoint_t* cp);
esp_err_t pbft_send_state_request(const pbft_state_request_t* req);
esp_err_t pbft_send_state_response(const pbft_state_response_t* resp);
```

Each `pbft_send_*` function:

1. Serializes the struct to wire format (packed, little-endian).
2. Computes HMAC over the MAC input (CRYPTO.md §5.1) using the appropriate HMAC key (broadcast = primary's key for outgoing, peer-specific key for incoming).
3. Dispatches via the selected transport (PROTOCOL.md §2). **No runtime fallback** — transport is chosen at compile time (see [PROTOCOL.md §8.1](./PROTOCOL.md#81-solutions-for-oversized-messages) for the constraint rationale).
4. Returns `ESP_OK` on enqueue (delivery is async — send_cb fires later).

`pbft_resend_*` variants are used by the timeout/retry logic in CONSENSUS.md §6.3; they re-send the **same message** (no new MAC needed because the MAC is over the message content which is unchanged).

### 15.3 Dispatch (PROTOCOL.md §11)

```c
// pbft_dispatch.h
void pbft_network_dispatch(const uint8_t* buf, size_t len, uint8_t sender_id);
```

Called by the transport rx callback. Extracts `msg_type` from the header, dispatches to the right handler:

| msg_type | Handler |
|----------|---------|
| 0 (HELLO) | `pbft_on_hello` (CRYPTO §6.2) |
| 1 (PRE_PREPARE) | `pbft_handle_pre_prepare` (CONSENSUS §4.2) |
| 2 (PREPARE) | `pbft_handle_prepare` (CONSENSUS §4.3) |
| 3 (COMMIT) | `pbft_handle_commit` (CONSENSUS §4.4) |
| 4 (VIEW_CHANGE) | `pbft_handle_view_change` (VIEW-CHANGE §5.1) |
| 5 (NEW_VIEW) | `pbft_handle_new_view` (VIEW-CHANGE §5.3) |
| 6 (CHECKPOINT) | `pbft_handle_checkpoint` (CHECKPOINT §5.4) |
| 7 (STATE_REQUEST) | `pbft_handle_state_request` (CHECKPOINT §8) |
| 8 (STATE_RESPONSE) | `pbft_handle_state_response` (CHECKPOINT §8) |

### 15.4 Dedup cache (CONSENSUS.md §15)

```c
// pbft_dedup.h
#define PBFT_DEDUP_CACHE_SIZE 256   // entries; entry = {digest[32], sequence:u64} = 40 B; ~10 KB; FIFO eviction

esp_err_t pbft_dedup_check_and_insert(const uint8_t digest[32],
                                       uint64_t* new_seq,
                                       uint32_t now_ms);
```

The dedup / executed-digest cache holds `PBFT_DEDUP_CACHE_SIZE = 256` entries (each `{digest[32], sequence:u64}` = 40 B, ~10 KB total, FIFO eviction). It is present on **EVERY node** — not primary-only:

- **Primary** (on `pbft_submit`): if the TX digest is already cached, the call returns the **existing** sequence (collapses duplicate submissions) instead of assigning a new one.
- **Replica** (on Pre-Prepare): if the Pre-Prepare's digest is cached with a sequence `<= high_watermark` (i.e. already committed/executed), the replica logs an alarm and **rejects** the Pre-Prepare (Byzantine-primary replay defense).

`pbft_dedup_check_and_insert` returns `PBFT_ERR_DUPLICATE` if the digest is already in the cache; the existing sequence is returned via the `new_seq` out-parameter (caller can re-use or reject). New-View carries executed digests so a fresh primary pre-populates this cache.

### 15.5 Misc internal helpers

```c
// pbft_view_change.h — fetch the stored View-Change message for a view from NVS
const pbft_view_change_t* pbft_nv_get_vc(uint32_t view);

// pbft_checkpoint.h — find a stable-checkpoint proof for a given sequence (NULL if none)
const pbft_checkpoint_proof_t* pbft_checkpoint_proof_find(uint64_t sequence);

// pbft_util.h — single-byte popcount over one quorum bitmask (prepare/commit/exec)
uint8_t pbft_count_set_bits(uint8_t bitmask);
```

| Function | Purpose | Used by |
|----------|---------|---------|
| `pbft_nv_get_vc` | Retrieve the persisted View-Change record for `view`. | VIEW-CHANGE.md §6 |
| `pbft_checkpoint_proof_find` | Locate a stable-checkpoint proof covering `sequence`; returns NULL if none. | CHECKPOINT.md §7 |
| `pbft_count_set_bits` | Popcount over a **single-byte** quorum bitmask (`prepare_bitmask` / `commit_bitmask` / `exec_bitmask`). Replaces the removed array-form `pbft_count_set_bits_from_array`. | CONSENSUS.md / VIEW-CHANGE.md quorum checks |

---

## 16. References

- [HANDOVER.md](./HANDOVER.md) — high-level API preview
- [CONSENSUS.md](./CONSENSUS.md) — `pbft_submit`, log handlers, dedup
- [CHECKPOINT.md](./CHECKPOINT.md) — `pbft_state_digest_cb`, GC, state transfer
- [VIEW-CHANGE.md](./VIEW-CHANGE.md) — internal view-change API (not public)
- [PROTOCOL.md](./PROTOCOL.md) — message types, transport
- [CRYPTO.md](./CRYPTO.md) — Y-5 re-gen API, MAC compute/verify
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — A3, C3, C4, C5, D3, D4, D5

---

**End of API-REFERENCE.md (v0.2 — addresses audit issues A3, C3, C4, C5 + nits D3, D4, D5)**