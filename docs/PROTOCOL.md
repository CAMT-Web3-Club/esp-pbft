# esp-pbft — Protocol & Wire Format

> **Status:** 🟡 Draft
> **Scope:** Wire format, transport abstraction, byte layout, addressing
> **Out of scope:** Crypto internals → [CRYPTO.md](./CRYPTO.md), consensus state machine → [CONSENSUS.md](./CONSENSUS.md)

---

## 1. Overview

esp-pbft messages are exchanged over a **transport-agnostic abstraction** with two compile-time implementations:
- **ESP-NOW** (default) — peer-to-peer broadcast, no AP
- **Wi-Fi UDP multicast** — AP + IGMP group

All messages share a **common wire format** regardless of transport. Switching transport at Kconfig time does NOT change the PBFT protocol semantics.

---

## 2. Transport abstraction (`pbft_network.h`)

### 2.1 Interface

```c
// pbft_network.h
typedef enum {
    PBFT_NET_OK              = 0,
    PBFT_NET_ERR_TIMEOUT     = -1,
    PBFT_NET_ERR_INVALID     = -2,
    PBFT_NET_ERR_FULL        = -3,
} pbft_net_err_t;

typedef struct {
    uint8_t  my_node_id;                  // 0..6
    uint32_t broadcast_timeout_ms;        // default 1000
    uint8_t  retry_count;                 // default 3
} pbft_net_config_t;

// Transport ops (function pointer table)
typedef struct {
    const char* name;                      // "esp-now" / "wifi-udp"
    pbft_net_err_t (*init)(const pbft_net_config_t* cfg);
    pbft_net_err_t (*deinit)(void);
    pbft_net_err_t (*broadcast)(const void* buf, size_t len);
    pbft_net_err_t (*unicast)(uint8_t peer_id, const void* buf, size_t len);
    bool (*is_ready)(void);
} pbft_transport_t;

// Select transport at compile time
#if defined(CONFIG_PBFT_TRANSPORT_ESP_NOW)
extern const pbft_transport_t* pbft_get_transport(void);  // returns &pbft_transport_espnow
#elif defined(CONFIG_PBFT_TRANSPORT_WIFI_UDP)
extern const pbft_transport_t* pbft_get_transport(void);  // returns &pbft_transport_wifi_udp
#else
#error "Must define CONFIG_PBFT_TRANSPORT_ESP_NOW or CONFIG_PBFT_TRANSPORT_WIFI_UDP"
#endif
```

### 2.2 ESP-NOW implementation (`pbft_network_espnow.c`)

**API used:** `esp_now_send()`, `esp_now_register_send_cb()`, `esp_now_add_peer()`

```c
static pbft_net_err_t espnow_init(const pbft_net_config_t* cfg) {
    esp_now_init();
    esp_now_register_send_cb(espnow_send_cb);
    // Register all 6 peers as broadcast-capable
    for (int i = 0; i < PBFT_CLUSTER_SIZE; i++) {
        if (i == cfg->my_node_id) continue;
        esp_now_peer_info_t peer = {
            .peer_addr = PBFT_ESP_NOW_BROADCAST_MAC,   // all-FF
            .channel = 0,
            .encrypt = false,  // HMAC at PBFT layer, CCMP at ESP-NOW
        };
        esp_now_add_peer(&peer);
    }
    return PBFT_NET_OK;
}

static pbft_net_err_t espnow_broadcast(const void* buf, size_t len) {
    if (len > 250) return PBFT_NET_ERR_INVALID;  // ESP-NOW limit
    return esp_now_send(PBFT_ESP_NOW_BROADCAST_MAC, buf, len) == ESP_OK
           ? PBFT_NET_OK : PBFT_NET_ERR_FULL;
}
```

**Size constraint:** ESP-NOW max payload = **250 bytes** (verified for ESP32-C3).

The `PBFT_ESP_NOW_BROADCAST_MAC` constant is defined in `pbft_network_espnow.h`:

```c
#define PBFT_ESP_NOW_BROADCAST_MAC   { 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF }
#define PBFT_ESP_NOW_MAX_PAYLOAD     250
#define PBFT_UDP_MTU                 1500
#define PBFT_UDP_MAX_PAYLOAD         1472
```

### 2.3 Wi-Fi UDP implementation (`pbft_network_wifi_udp.c`)

**API used:** `socket()`, `sendto()`, `setsockopt(IP_ADD_MEMBERSHIP)`, `esp_wifi_set_mode(WIFI_MODE_AP)`, `esp_wifi_set_ps()`

```c
#define PBFT_MCAST_GROUP  "239.42.0.7"   // site-local, scoped
#define PBFT_MCAST_PORT   7707

static pbft_net_err_t wifi_udp_init(const pbft_net_config_t* cfg) {
    // 1. Start Wi-Fi in AP mode
    esp_netif_create_default_wifi_ap();
    esp_wifi_set_mode(WIFI_MODE_AP);
    esp_wifi_set_config(/* SSID, channel, etc. */);
    esp_wifi_start();

    // 2. Apply power-save mode per role (see POWER.md §12)
    wifi_udp_set_role(cfg->my_node_id == 0);

    // 3. Create UDP socket
    int fd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(PBFT_MCAST_PORT),
        .sin_addr.s_addr = htonl(INADDR_ANY),
    };
    bind(fd, (struct sockaddr*)&addr, sizeof(addr));

    // 3. Join multicast group
    struct ip_mreq mreq = {
        .imr_multiaddr.s_addr = inet_addr(PBFT_MCAST_GROUP),
        .imr_interface.s_addr = htonl(INADDR_ANY),
    };
    setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));

    // 4. Map node_id → IP (config or DHCP lease table)
    //    node_id 0 = 192.168.4.1, node_id 1 = 192.168.4.2, etc.
    return PBFT_NET_OK;
}
```

**Size constraint:** UDP max payload = **65,507 bytes** (IP MTU 1500 in practice → ~1472 bytes).

**Power-save modem mode (mandatory):** AP node → `WIFI_PS_NONE`; STA nodes → `WIFI_PS_MIN_MODEM`. Without this, Wi-Fi UDP build draws ~160 mA continuously; with `WIFI_PS_MIN_MODEM` + DFS ON + DTIM 1 + 160 MHz, STA current drops to **11.35 mA average** (Espressif-measured, [wifi-performance-and-power-save.html](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/wifi-driver/wifi-performance-and-power-save.html)). Full rationale, Kconfig, and edge cases in [POWER.md §12](./POWER.md#12-wi-fi-udp-power-save-modem-mode).

### 2.5 UDP multicast vs unicast fan-out — AP multicast-cache limitation

> ⚠️ **Espressif hardware limitation (verified, all targets):** "Currently, ESP32 AP does not support all of the power-saving feature defined in Wi-Fi specification. To be specific, **the AP only caches unicast data for the stations** connect to this AP, **but does not cache the multicast data for the stations**. If stations connected to the ESP32 AP are power-saving enabled, they may experience **multicast packet loss**."
> — [Espressif Wi-Fi power-save guide](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/wifi-driver/wifi-performance-and-power-save.html), applies verbatim to ESP32 / S2 / S3 / S31 / C2 / C3 / C5 / C6 / C61.

**Consequence for esp-pbft:** if we use **group UDP multicast** for Prepare/Commit, STAs in modem-sleep (the entire esp-pbft power-saving design) will lose messages. This is a **P0 liveness bug** — quorum (2f+1 = 5 of 7) may not be reached even with f=0 faulty replicas.

**Mitigation chosen:** UDP transport uses **unicast-fan-out** for Prepare/Commit (one `sendto()` per peer, collecting ACKs into the quorum set). ESP-NOW transport keeps broadcast (no AP involved, no multicast-cache limitation). The `pbft_transport_t.unicast()` hook in §2.1 already supports this; no transport-API change required.

**Kconfig:**

```kconfig
config PBFT_UDP_FANOUT_MODE_UNICAST
    bool "Wi-Fi UDP: unicast fan-out (recommended)"
    default y
    help
      Sends Prepare and Commit via 6 unicast UDP packets (one per peer).
      This is MANDATORY for power-saving: Espressif AP firmware does not
      cache multicast frames for sleeping STAs, so multicast Prepare/Commit
      would be lost under WIFI_PS_MIN_MODEM.
      See PROTOCOL.md §2.5 and Espressif wifi-performance-and-power-save.html.
```

### 2.4 Common receive callback signature

Both transports dispatch received messages via a single callback:

```c
typedef void (*pbft_net_rx_cb_t)(const uint8_t* buf, size_t len, uint8_t sender_id);

void pbft_network_register_rx_callback(pbft_net_rx_cb_t cb);
```

The `sender_id` is extracted from the packet header (§4), NOT from MAC/IP — this makes the transport truly interchangeable.

---

## 3. Transport comparison

| Aspect | ESP-NOW | Wi-Fi UDP multicast |
|--------|---------|----------------------|
| **Max payload** | 250 B | ~1472 B (MTU) |
| **Max nodes (theoretical)** | 20 (encrypt) / 6 (unencrypt) | unlimited |
| **AP needed** | ❌ no | ✅ yes |
| **Latency** | 10-20 ms | 20-50 ms |
| **Power** | 80 mA | 160 mA |
| **Cluster size fit (n=7)** | ✅ perfect | ✅ overkill but works |
| **Address** | MAC (6 B) | IPv4 (4 B) + port |
| **Transport header** | 24 B (802.11) | 28-54 B (IP+UDP) |
| **Default** | ✅ chosen | — |
| **PBFT Prepare/Commit** | broadcast | **unicast-fan-out** (see §2.5) |

**Decision rationale:** see [HANDOVER.md §3.3](./HANDOVER.md).

---

## 4. Wire format (common header)

All PBFT messages start with a **4-byte common header**, followed by message-type-specific fields.

### 4.1 Common header (4 bytes)

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  msg_type     |  sender_id    |          reserved             |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  msg_type  : uint8  = PBFT_MSG_* (see §5)
  sender_id : uint8  = 0..6 (PBFT_CLUSTER_SIZE)
  reserved  : uint16 = 0 (MBZ; for future use)
```

**Total common header:** 4 bytes. Little-endian for multi-byte fields.

### 4.2 Endianness

All multi-byte fields are **little-endian** (ESP32-C3 is RISC-V, native LE).

### 4.3 Alignment

- Header is 4-byte aligned (4 bytes, naturally aligned).
- `view` (4 B) and `sequence` (8 B) below are 8-byte aligned when preceded by 4-byte header — compiler may insert 4 B padding for some message types. We do not require natural alignment of all fields (would waste 4 B per packet).

---

## 5. Message types

| Type ID | Name | Direction | Size (typical) | Defined in |
|---------|------|-----------|----------------|------------|
| 0 | `PBFT_MSG_HELLO` | bidirectional (bootstrap) | 4 + 1 + 65 + 32 = **102 B wire, 104 B with 2 B alignment pad** | §6.1 |
| 1 | `PBFT_MSG_PRE_PREPARE` | primary → all | 4 + 4 + 8 + 32 + 2 + payload + 32 = **82 + payload B** | §6.2 |
| 2 | `PBFT_MSG_PREPARE` | all → all | 4 + 4 + 8 + 32 + 32 = **80 B** | §6.3 |
| 3 | `PBFT_MSG_COMMIT` | all → all | 4 + 4 + 8 + 32 + 32 = **80 B** | §6.4 |
| 4 | `PBFT_MSG_VIEW_CHANGE` | all → all | 4 + 88 + n_prepared × 40 = **88–4088 B** | §6.5 + [VIEW-CHANGE.md §4.3](./VIEW-CHANGE.md) |
| 5 | `PBFT_MSG_NEW_VIEW` | new primary → all | var (always UDP) ≈ **2.5–6 KB** | §6.6 + [VIEW-CHANGE.md §4.4](./VIEW-CHANGE.md) |
| 6 | `PBFT_MSG_CHECKPOINT` | all → all | 4 + 8 + 32 + 32 = **76 B** | §6.7 + [CHECKPOINT.md §5](./CHECKPOINT.md) |
| 7 | `PBFT_MSG_STATE_REQUEST` | replica → peer | 4 + 4 + 8 + 32 = **48 B** | [CHECKPOINT.md §8](./CHECKPOINT.md) |
| 8 | `PBFT_MSG_STATE_RESPONSE` | replica → requesting peer | 4 + 4 + 8 + var + 32 = **48+ B** | [CHECKPOINT.md §8](./CHECKPOINT.md) |

Reserved: 9-255 (MBZ).

All messages except Hello include a 32-byte HMAC trailer (see §6).

---

## 6. Message details

### 6.1 Hello (`PBFT_MSG_HELLO`, type 0)

Used for cluster discovery + soft re-handshake (Pattern Y-5).

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |        common header (msg_type=0, sender_id, reserved)        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                                                               |
  |          pubkey (65 bytes: 0x04 ‖ X_BE(32) ‖ Y_BE(32))        |
  |                                                               |
  |                                                               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                  re_handshake (1 byte)                        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                  mac (32 bytes, HMAC-SHA256)                  |
  |                                                               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  Total: 4 + 65 + 1 + 32 = 102 bytes (+ 2 padding = 104 if needed)
```

| Field | Size | Notes |
|-------|------|-------|
| `msg_type` | 1 B | = 0 (PBFT_MSG_HELLO) |
| `sender_id` | 1 B | 0..6 |
| `reserved` | 2 B | MBZ |
| `pubkey` | 65 B | uncompressed P-256: `0x04` (1 B) ‖ `X` (32 B, big-endian) ‖ `Y` (32 B, big-endian). Matches `psa_export_public_key()` output verbatim and `psa_import_key()` requirement for `PSA_KEY_TYPE_ECC_PUBLIC_KEY(SECP_R1)`. |
| `re_handshake` | 1 B | 0 = first hello, 1 = re-handshake marker |
| `mac` | 32 B | HMAC-SHA256 over (common header ‖ pubkey ‖ re_handshake) using current `hmac_keys[sender_id]` if known, or zero-filled if first hello (TOFU) |

**Encoding note:** To avoid padding, structure is `__attribute__((packed))` and serialized manually. The `reserved` field is included so the MAC input starts at a 4-byte boundary.

**PSA wire format reference:** Verified against esp-idf master `components/mbedtls/test_apps/.../test_psa_ecdsa.c:700-705`, `components/bt/esp_ble_mesh/common/crypto_psa.c:419-432`, `components/wpa_supplicant/esp_supplicant/src/crypto/crypto_mbedtls-ec.c:1714-1822`, `components/bt/controller/esp32c6/bt.c:1699-1740`. If a peer sends 33-byte compressed (0x02/0x03 ‖ X), the receiver must rehydrate to 65-byte uncompressed before calling `psa_import_key()`.

### 6.2 Pre-Prepare (`PBFT_MSG_PRE_PREPARE`, type 1)

Primary assigns sequence number and broadcasts TX.

```
   0                   1                   2                   3
   0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |        common header (msg_type=1, sender_id, reserved)        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                       view (uint32)                          |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                   sequence (uint64)                           |
  |                                                               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                   digest (32 bytes)                           |
  |                                                               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |        payload_len (uint16)  |  payload (variable, 0-256 B)  |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                                                               |
  |                   mac (32 bytes)                              |
  |                                                               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  Total: 4 + 4 + 8 + 32 + 2 + payload + 32 = 82 + payload bytes
```

| Field | Size | Notes |
|-------|------|-------|
| common header | 4 B | type=1 |
| `view` | 4 B | current view number |
| `sequence` | 8 B | monotonic TX order position |
| `digest` | 32 B | SHA-256 of TX payload (allows prepare/commit without full payload) |
| `payload_len` | 2 B | 0-256 |
| `payload` | 0-256 B | actual TX bytes |
| `mac` | 32 B | HMAC over (common header ‖ view ‖ seq ‖ digest ‖ payload) |

**Padding:** if `payload_len` is odd, insert 1 B pad to keep `mac` at 4-byte boundary (saves 1 padding B elsewhere).

### 6.3 Prepare (`PBFT_MSG_PREPARE`, type 2)

Replica agrees on (view, sequence, digest) but does not echo TX payload.

```
   common header (type=2) | view (4) | sequence (8) | digest (32) | mac (32)
   Total: 4 + 4 + 8 + 32 + 32 = 80 bytes
```

### 6.4 Commit (`PBFT_MSG_COMMIT`, type 3)

Replica confirms it has reached `prepared` state.

```
   Same layout as Prepare: 80 bytes
```

### 6.5 View-Change (`PBFT_MSG_VIEW_CHANGE`, type 4)

Sent when replica suspects primary is faulty.

**Canonical wire format is in [VIEW-CHANGE.md §4.3](./VIEW-CHANGE.md)** — includes the **prepared-set** (`n_prepared : uint8` + `n_prepared × {seq:u64, digest:32}`) required for the new primary to construct the O-set. The summary here is a preview; defer to VIEW-CHANGE.md for byte-exact layout.

```
   common header (type=4) | view (4) | new_view (4) | checkpoint_seq (8) |
   checkpoint_digest (32) | n_prepared (1) | reserved (3) |
   {prepared_entry: n_prepared × (8 + 32)} | mac (32)

   Empty prepared-set: 4 + 4 + 4 + 8 + 32 + 1 + 3 + 32 = 88 bytes
   Worst case (n_prepared=100): 88 + 100 × 40 = 4088 bytes → UDP only
```

| Field | Size | Notes |
|-------|------|-------|
| `view` | 4 B | current view (suspected-faulty primary's view) |
| `new_view` | 4 B | next view to transition to |
| `checkpoint_seq` | 8 B | last stable checkpoint sequence number |
| `checkpoint_digest` | 32 B | digest of app state at checkpoint |
| `n_prepared` | 1 B | number of prepared entries (0..PBFT_VC_MAX_PREPARED, default 10) |
| prepared entries | 0–4000 B | each is `{seq:u64, digest:32}` = 40 B |
| `mac` | 32 B | HMAC over header + body (no MAC trailer on entries) |

**Kconfig:**

```kconfig
config PBFT_VC_MAX_PREPARED
    int "Max prepared entries carried in a View-Change"
    default 10
    range 0 100
    help
      Caps the worst-case View-Change size. With n_prepared=10 the
      worst-case VC = 488 B (still fits ESP-NOW); with 100 it's 4088 B
      (forces UDP). Set per cluster reliability requirements.
```

### 6.6 New-View (`PBFT_MSG_NEW_VIEW`, type 5)

New primary broadcasts its proof + new Pre-Prepare messages.

**Canonical wire format is in [VIEW-CHANGE.md §4.4](./VIEW-CHANGE.md)** — uses **split MACs** (each embedded Pre-Prepare carries its own Pre-Prepare MAC; New-View outer MAC binds `view ‖ num_vc ‖ v_set_digest`). The summary here is a preview.

```
   common header (type=5) | view (4) | num_vc (1) | reserved (3) |
   v_set_digest (32) |
   { VC proof: num_vc × pbft_vc_size_t } |
   { O-set Pre-Prepares: each carries its own Pre-Prepare MAC } |
   mac (32)              ← over header + view + num_vc + v_set_digest + VC proofs

   Worst case (n=7, n_prepared=10 each VC): ≈ 2516 B
   + O-set (10 Pre-Prepares × 338 B): ≈ 3380 B
   Grand total worst case ≈ 5900 B → UDP only
```

| Field | Size | Notes |
|-------|------|-------|
| `view` | 4 B | new view number (v+1) |
| `num_vc` | 1 B | number of VIEW-CHANGE proofs (≥2f+1 = 5) |
| `v_set_digest` | 32 B | SHA-256 over concatenation of all VC MACs |
| VC proofs | var | embedded VIEW-CHANGE messages (each with own MAC) |
| O-set | var | Pre-Prepares to re-propose (each with own MAC) |
| `mac` | 32 B | HMAC over the rest (excluding O-set) |

**Size concern:** worst-case ~5.9 KB. **Always UDP** — see §8.2.

### 6.7 Checkpoint (`PBFT_MSG_CHECKPOINT`, type 6)

```
   common header (type=6) | sequence (8) | app_state_digest (32) | mac (32)
   Total: 4 + 8 + 32 + 32 = 76 bytes (+ 4 padding = 80)
```

| Field | Size | Notes |
|-------|------|-------|
| `sequence` | 8 B | sequence number of checkpoint |
| `app_state_digest` | 32 B | SHA-256 of app state at this checkpoint |
| `mac` | 32 B | HMAC |

---

## 7. MAC computation (recap from CRYPTO.md)

For every message except Hello-initial, MAC input = `view ‖ sequence ‖ msg_type ‖ digest ‖ payload`. Full details in [CRYPTO.md §5](./CRYPTO.md#5-per-message-mac).

---

## 8. ESP-NOW size constraint (250 B) — implications

| Message | Size | Fits in ESP-NOW? |
|---------|------|------------------|
| Hello | 104 B (102 B wire + 2 B pad to align) | ✅ |
| Pre-Prepare (256 B payload) | 338 B | ❌ exceeds |
| Prepare | 80 B | ✅ |
| Commit | 80 B | ✅ |
| View-Change | 88 B | ✅ |
| New-View | ~500 B | ❌ exceeds |
| Checkpoint | 80 B | ✅ |

### 8.1 Solutions for oversized messages

**Option A: Wi-Fi UDP for oversized messages** (recommended)
- ESP-NOW for normal 3-phase messages
- Wi-Fi UDP for New-View (and Pre-Prepare with >250 B payload)
- Application decides per-message which transport to use

**Option B: Fragmentation** (complex)
- Split large messages across multiple ESP-NOW packets
- Adds reassembly code, sequence numbers, timeouts
- Not recommended for v1

**Option C: Cap TX payload size**
- Reduce `PBFT_TX_PAYLOAD_MAX` to ~150 B
- New-View still too big — need UDP anyway
- Not recommended

**Decision:** **Option A** — esp-pbft uses **Wi-Fi UDP for New-View** even when ESP-NOW is the default transport for normal messages. This is documented in the transport abstraction.

### 8.2 Dual-transport runtime

```c
esp_err_t pbft_send_pre_prepare(const pbft_msg_t* msg) {
    size_t total_len = msg->payload_len + 82;  // 82 = header overhead
    if (total_len > 250 && pbft_default_transport == &pbft_transport_espnow) {
        // Fallback to UDP for oversized messages
        return udp_transport.broadcast(msg, total_len);
    }
    return pbft_default_transport->broadcast(msg, total_len);
}

esp_err_t pbft_send_new_view(const pbft_new_view_t* nv) {
    // Always use UDP — New-View doesn't fit in ESP-NOW
    return udp_transport.broadcast(nv, sizeof(*nv));
}
```

This requires both transports to be **initialized at boot** even if ESP-NOW is "default" (additional ~30 KB RAM).

---

## 9. Node addressing

### 9.1 Node ID → MAC mapping (ESP-NOW)

Node ID is assigned at factory provisioning (stored in NVS or hard-coded):

```c
// NVS key "node_id" → uint8_t
uint8_t my_node_id = nvs_get_u8("node_id", 0);

// ESP-NOW uses MAC address — but for broadcast, MAC = FF:FF:FF:FF:FF:FF
// So node_id is embedded in PACKET, not derived from MAC
```

This means **all nodes receive all messages** (broadcast), and `sender_id` field in the header identifies the source.

### 9.2 Node ID → IP mapping (Wi-Fi UDP)

```c
// node_id 0 = 192.168.4.1 (AP gateway)
// node_id 1 = 192.168.4.2 (STA client 1)
// node_id 2 = 192.168.4.3 (STA client 2)
// ...
// Or use DHCP lease table to map IP ↔ node_id
```

**Recommendation:** in AP mode, node 0 is the AP itself (`192.168.4.1`), other nodes are DHCP clients. Node ID stored in NVS.

### 9.3 Cluster config (NVS)

Each node stores the **complete cluster config** in NVS at provisioning time:

```
# NVS namespace "pbft_cluster" — written once at factory
key "node_id_0" → MAC addr or IP addr (depending on transport)
key "node_id_1" → ...
...
key "my_node_id" → 0..6
```

This is **not** the same as the public-key cache (TOFU) — see [CRYPTO.md §6](./CRYPTO.md#6-discovery).

---

## 10. Packet lifecycle (send path)

```
   Consensus state machine decides to send msg X
                │
                ▼
   1. pbft_crypto_compute_mac(msg, peer_or_broadcast)
      → fills msg.mac[32] (5 µs)
                │
                ▼
   2. pbft_network_send(peer_id, msg, len)
      → dispatches to transport
                │
                ├─── ESP-NOW ─── esp_now_send() ─── 802.11 broadcast
                │
                └─── UDP ────── sendto(mcast_addr) ─── IGMP
                │
                ▼
   3. (async) send_cb fires
      → update metrics, log errors
```

---

## 11. Packet lifecycle (receive path)

```
   802.11 broadcast or UDP multicast
                │
                ▼
   Transport rx callback fires (registered at init)
                │
                ▼
   pbft_network_dispatch(buf, len, sender_id)
      → extracts sender_id from header
      → if len < 4 || msg_type invalid → drop
                │
                ▼
   pbft_crypto_verify_mac(msg, sender_id)
      → re-compute + constant-time compare (~5 µs)
      → if invalid → drop (log)
                │
                ▼
   pbft_consensus_handle(msg)
      → state machine: IDLE → PRE_PREPARED → ...
```

---

## 12. Constraints summary

| Constraint | Value | Source |
|------------|-------|--------|
| `PBFT_CLUSTER_SIZE` | 7 | HANDOVER §3.2 |
| `PBFT_TX_PAYLOAD_MAX` | 256 B | Kconfig |
| ESP-NOW max payload | 250 B | ESP-IDF verified |
| UDP MTU | 1500 B | standard |
| Common header | 4 B | this doc |
| HMAC | 32 B | SHA-256 |
| ECDSA pubkey (uncompressed) | 65 B | secp256r1 (PSA `psa_export_public_key` returns 0x04 ‖ X_BE(32) ‖ Y_BE(32)) |
| View | u32 | PBFT |
| Sequence | u64 | PBFT (large for long-lived) |
| Min packet size | 76 B (Checkpoint) | this doc |
| Max packet size (Pre-Prepare) | 82 + 256 = 338 B | exceeds ESP-NOW |
| Max packet size (New-View) | ~500 B | exceeds ESP-NOW → must use UDP |

---

## 13. Open questions (protocol-level)

| # | Question | Status |
|---|----------|--------|
| P1 | Should we use Wi-Fi UDP by default given New-View constraint? | 🟡 Open |
| P2 | ESP-NOW fragmentation worth it? | 🟡 No (UDP fallback simpler) |
| P3 | Should we add message-type versioning? | 🟡 Open (future) |
| P4 | Compress payload for Pre-Prepare? | 🟡 No (overhead > savings at 256 B) |

---

## 14. References

- [HANDOVER.md](./HANDOVER.md) — n=7, f=2, transport selection
- [ARCHITECTURE.md](./ARCHITECTURE.md) §2 boot, §3 consensus flow
- [CRYPTO.md](./CRYPTO.md) §5 MAC computation, §6 Hello
- [CODING_STANDARD.md](./CODING_STANDARD.md) §3 anti-pattern A4 (pointer aliasing)
- ESP-IDF ESP-NOW docs: https://docs.espressif.com/projects/esp-idf/en/v6.0.1/esp32c3/api-reference/network/esp-now.html
- RFC 768 — UDP
- RFC 1112 — IP multicast / IGMP

---

**End of PROTOCOL.md (v0.1 draft — awaiting review)**