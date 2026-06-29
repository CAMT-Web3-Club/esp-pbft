# esp-pbft — Protocol & Wire Format

> **Status:** 🟡 Draft
> **Scope:** Wire format, transport abstraction, byte layout, addressing
> **Out of scope:** Crypto internals → [CRYPTO.md](./CRYPTO.md), consensus state machine → [CONSENSUS.md](./CONSENSUS.md)

> ⚠️ **CRITICAL DESIGN CONSTRAINT (no runtime fallback):** esp-pbft does **NOT**
> fall back from ESP-NOW to Wi-Fi UDP at runtime. The transport is a
> **compile-time** choice only, via the mutually-exclusive `choice PBFT_TRANSPORT`
> Kconfig block: either `CONFIG_PBFT_TRANSPORT_ESP_NOW=y` or
> `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y`. There is **no** "ESP-NOW with UDP fallback"
> or "dual-transport runtime" mode. An oversized message on the selected transport
> returns `PBFT_NET_ERR_INVALID` at runtime and the build's `_Static_assert`s
> prevent this at compile time (see [MEMORY.md §X](./MEMORY.md)). See §8.1 below
> for the rationale and the consequences for New-View support.

---

## 1. Overview

esp-pbft messages are exchanged over a **transport-agnostic abstraction** with two compile-time implementations:
- **ESP-NOW** (default) — peer-to-peer broadcast, no AP
- **Wi-Fi UDP multicast** — AP + IGMP group

All messages share a **common wire format** regardless of transport. Switching transport at Kconfig time does NOT change the PBFT protocol semantics. **The transport choice is fixed at compile time and cannot be changed at runtime.**

---

## 2. Transport abstraction (`pbft_network.h`)

### 2.1 Interface

```c
// pbft_network.h
// Transport-internal status enum. Renamed from the old `pbft_net_err_t` to avoid
// clashing with API-REFERENCE's public `pbft_err_t`. The transport layer maps
// pbft_transport_status_t -> pbft_err_t (PBFT_NET_ERR_TIMEOUT=-50, _INVALID=-51,
// _FULL=-52, _NO_ROUTE=-53). See API-REFERENCE §3.
typedef enum {
    PBFT_TX_OK      =  0,
    PBFT_TX_TIMEOUT = -1,
    PBFT_TX_INVALID = -2,
    PBFT_TX_FULL    = -3,
} pbft_transport_status_t;

typedef struct {
    uint8_t  my_node_id;                  // 0..6
    uint32_t broadcast_timeout_ms;        // default 1000
    uint8_t  retry_count;                 // default 3
} pbft_net_config_t;

// Transport ops (function pointer table)
typedef struct {
    const char* name;                      // "esp-now" / "wifi-udp"
    pbft_transport_status_t (*init)(const pbft_net_config_t* cfg);
    pbft_transport_status_t (*deinit)(void);
    pbft_transport_status_t (*broadcast)(const void* buf, size_t len);
    pbft_transport_status_t (*unicast)(uint8_t peer_id, const void* buf, size_t len);
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
static pbft_transport_status_t espnow_init(const pbft_net_config_t* cfg) {
    esp_now_init();
    esp_now_register_send_cb(espnow_send_cb);
    // AUDIT-FIX SEC-Y-13 (2026-06-29): enable per-peer CCMP encryption.
    // The previous `.encrypt = false` disabled air-layer integrity so an
    // attacker with a $5 ESP32 + 30 lines of Arduino could jam the channel
    // or replay-storm the cluster (no MAC check at air layer). ESP-NOW CCMP
    // (WPA2-PSK-style) is built-in for encrypted peers; ESP-NOW supports up
    // to 20 encrypted peers — sufficient for n=7. The PTK is derived from a
    // per-cluster pre-shared key (provisioned alongside `my_node_id`); the
    // PMK is held in NVS.
    //
    // Broadcast still goes to FF:FF:FF:FF:FF:FF but with the air-layer MAC
    // enabled. CCMP costs ~1 µs per packet (AES-CCM HW on ESP32-C3).
    for (int i = 0; i < PBFT_CLUSTER_SIZE; i++) {
        if (i == cfg->my_node_id) continue;
        esp_now_peer_info_t peer = {
            .peer_addr = PBFT_ESP_NOW_BROADCAST_MAC,   // all-FF
            .channel = 0,
            .encrypt = true,                              // AUDIT-FIX SEC-Y-13
            .pmk = s_pmk,                                 // 16 B; loaded from NVS at boot
        };
        esp_now_add_peer(&peer);
    }
    // Initialise per-source-MAC airtime rate limiter (§2.2a) and RSSI floor.
    pbft_airtime_limiter_init();
    return PBFT_TX_OK;
}

static pbft_transport_status_t espnow_broadcast(const void* buf, size_t len) {
    if (len > PBFT_ESP_NOW_MAX_PAYLOAD) return PBFT_TX_INVALID;  // ESP-NOW limit
    return esp_now_send(PBFT_ESP_NOW_BROADCAST_MAC, buf, len) == ESP_OK
           ? PBFT_TX_OK : PBFT_TX_FULL;
}
```

**Size constraint:** ESP-NOW max payload = **250 bytes** for v1.0 (`ESP_NOW_MAX_IE_DATA_LEN` = 250) or **1470 bytes** for v2.0 (`ESP_NOW_MAX_DATA_LEN_V2` = 1470, defined in `esp_now.h` since ESP-IDF v6.0.1).

For esp-pbft v1 we use the v1.0 limit (250 B). View-Change (`PBFT_VC_MAX_PREPARED=4` → 248 B) fits; Pre-Prepare with 256 B payload (338 B) is sent over UDP. New-View (~1.3-4.7 KB) is always UDP.

The `PBFT_ESP_NOW_BROADCAST_MAC` constant is defined in `pbft_network_espnow.h`:

```c
#define PBFT_ESP_NOW_BROADCAST_MAC   { 0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF }
#define PBFT_ESP_NOW_MAX_PAYLOAD     250
#define PBFT_UDP_MTU                 1500
#define PBFT_UDP_MAX_PAYLOAD         1472
```

### 2.2a Airtime rate-limiter + RSSI sanity check (AUDIT-FIX SEC-Y-13, 2026-06-29)

Two RX-side guards against an attacker who holds airtime near the cluster.
Even with CCMP enabled (which authenticates the air frame), an attacker
can flood authenticated frames from a node whose PMK they have somehow
obtained, or simply jam. The cluster must still make progress.

```c
// (1) Per-source-MAC sliding-window rate limiter.
//     ESP-NOW headers carry the source MAC. PBFT worst-case legitimate
//     rate at 14 TPS is ~3 frames per TX (Pre-Prepare + Prepare + Commit) ×
//     6 senders = 18 frames per TX cycle, ≤ 30 frames/sec at peak. We set
//     the cap to PBFT_AIRTIME_MAX_PER_SEC (default 50) — 1.7× the worst
//     case, leaving headroom but blocking >50 frames/sec/sender which is
//     unambiguously an attack.
#define PBFT_AIRTIME_MAX_PER_SEC  50
#define PBFT_AIRTIME_WINDOW_MS   1000

static struct {
    uint8_t  mac[6];
    uint32_t window_start_ms;
    uint16_t count;
} s_airtime[7];

void pbft_airtime_limiter_init(void) {
    memset(s_airtime, 0, sizeof(s_airtime));
}

bool pbft_airtime_limiter_allow(const uint8_t src_mac[6]) {
    uint32_t now = now_ms();
    for (int i = 0; i < 7; i++) {
        if (memcmp(s_airtime[i].mac, src_mac, 6) != 0) continue;
        if (now - s_airtime[i].window_start_ms >= PBFT_AIRTIME_WINDOW_MS) {
            s_airtime[i].window_start_ms = now;
            s_airtime[i].count = 0;
        }
        if (s_airtime[i].count >= PBFT_AIRTIME_MAX_PER_SEC) return false;
        s_airtime[i].count++;
        return true;
    }
    // New source — register
    int empty = -1;
    for (int i = 0; i < 7; i++) if (s_airtime[i].mac[0] == 0) { empty = i; break; }
    if (empty < 0) return true;   // table full — fail open (rare)
    memcpy(s_airtime[empty].mac, src_mac, 6);
    s_airtime[empty].window_start_ms = now;
    s_airtime[empty].count = 1;
    return true;
}

// (2) RSSI floor — reject frames whose signal is too weak to be from a
//     cluster node. ESP-NOW exposes the RSSI of the last received frame
//     via the RX callback. PBFT is intended for physically co-located nodes;
//     -85 dBm is the floor for "same room" on a typical 2.4 GHz channel.
//     Frames below the floor are dropped (counted, then discarded). This
//     catches long-range replay attacks where an attacker captures a frame
//     and rebroadcasts it from outside the cluster's physical perimeter.
#define PBFT_RSSI_FLOOR_DBM  (-85)

bool pbft_rssi_acceptable(int8_t rssi_dbm) {
    return rssi_dbm >= PBFT_RSSI_FLOOR_DBM;
}
```

**Wire-up:** in `pbft_network_dispatch()` (PROTOCOL §11), reject frames that
fail either check before they reach the crypto/MAC verify path. Frames dropped
here are counted in `pbft_metrics_t.mac_failures` (existing metric, the same
counter is appropriate because the effect is the same from the consensus
viewpoint: the frame is unusable).

### 2.3 Wi-Fi UDP implementation (`pbft_network_wifi_udp.c`)

**API used:** `socket()`, `sendto()`, `setsockopt(IP_ADD_MEMBERSHIP)`, `esp_wifi_set_mode(WIFI_MODE_AP)`, `esp_wifi_set_ps()`

```c
#define PBFT_MCAST_GROUP  "239.42.0.7"   // site-local, scoped
#define PBFT_MCAST_PORT   7707

static pbft_transport_status_t wifi_udp_init(const pbft_net_config_t* cfg) {
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
    return PBFT_TX_OK;
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
| **Max nodes (theoretical)** | 20 total / 6 encrypted | unlimited |
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
  |  msg_type     |  sender_id    | proto_version |   reserved    |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

  msg_type      : uint8  = PBFT_MSG_* (see §5)
  sender_id     : uint8  = 0..6 (PBFT_CLUSTER_SIZE)
  proto_version : uint8  = PBFT_WIRE_VERSION (= 1)
  reserved      : uint8  = 0 (MBZ; for future use)
```

```c
// Wire-format version. Increment ONLY on a wire-incompatible change.
// v1 is frozen by this document.
#define PBFT_WIRE_VERSION   1

// Packed common header (4 bytes). memcpy-(de)serialized per CODING_STANDARD.
typedef struct __attribute__((packed)) {
    uint8_t msg_type;       // PBFT_MSG_*
    uint8_t sender_id;      // 0..6
    uint8_t proto_version;  // = PBFT_WIRE_VERSION
    uint8_t reserved;       // MBZ
} pbft_common_header_t;     // sizeof == 4
```

**Total common header:** 4 bytes. Little-endian for multi-byte fields.

**Wire-version check (receiver, mandatory):** every received packet MUST have
`proto_version == PBFT_WIRE_VERSION`; a packet whose `proto_version` differs is
**rejected** (counted as a dropped/invalid packet, same as a failed MAC). Setting
`proto_version` in the common header and enforcing this check on receive **freezes
the v1 wire format** — a future wire-incompatible change bumps `PBFT_WIRE_VERSION`
and old nodes will reject new-format packets rather than misparse them. See §11
(receive path) and §13 (open question P3). All nodes in a cluster MUST run the same
`PBFT_WIRE_VERSION` (rolling-OTA rule, cross-ref [DEPLOYMENT.md §5.2](./DEPLOYMENT.md)).

### 4.2 Endianness

- **Host-integer multi-byte fields** (view, sequence, lengths, timestamps) are **little-endian** (ESP32-C3 is RISC-V, native LE; no byte-swap needed at runtime).
- **Cryptographic fields** follow their standard canonical wire encoding per their respective specifications — they are NOT converted to host-endian:
  - **ECDSA P-256 public key coordinates** are **big-endian** per SEC1 / ANSI X9.62 uncompressed point format (`0x04 ‖ X_BE(32) ‖ Y_BE(32)`, total 65 bytes). This matches the output of PSA's `psa_export_public_key()` for P-256 ([esp-idf master](https://github.com/espressif/esp-idf/blob/master/components/wpa_supplicant/esp_supplicant/src/crypto/crypto_mbedtls-ec.c)).
  - **SHA-256 digests** are stored as raw byte arrays (no endianness concept — the bytes are the hash output).
  - **HMAC tags** are stored as raw byte arrays (same).

If a future wire field is added that uses a cryptographic format with mixed endianness (e.g., DSA signatures with r‖s where r,s may be DER-encoded), document the exception here with the source spec.

### 4.3 Alignment

- Header is 4-byte aligned (4 bytes, naturally aligned).
- `view` (4 B) and `sequence` (8 B) below are 8-byte aligned when preceded by 4-byte header — compiler may insert 4 B padding for some message types. We do not require natural alignment of all fields (would waste 4 B per packet).

---

## 5. Message types

| Type ID | Name | Direction | Size (typical) | Defined in |
|---------|------|-----------|----------------|------------|
| 0 | `PBFT_MSG_HELLO` | bidirectional (bootstrap) | 4 + 1 + 65 + 32 = **102 B wire, 104 B with 2 B alignment pad** | §6.1 |
| 1 | `PBFT_MSG_PRE_PREPARE` | primary → all | 4 (header) + 4 (view) + 8 (seq) + 32 (digest) + 2 (payload_len) + payload + 32 (mac) = **82 + payload B wire** | §6.2 |
| 2 | `PBFT_MSG_PREPARE` | all → all | 4 + 4 + 8 + 32 + 32 = **80 B** | §6.3 |
| 3 | `PBFT_MSG_COMMIT` | all → all | 4 + 4 + 8 + 32 + 32 = **80 B** | §6.4 |
| 4 | `PBFT_MSG_VIEW_CHANGE` | all → all | 4 + 88 + n_prepared × 40 = **88–248 B** (PBFT_VC_MAX_PREPARED=4, fits ESP-NOW) | §6.5 + [VIEW-CHANGE.md §4.3](./VIEW-CHANGE.md) |
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
| `proto_version` | 1 B | = PBFT_WIRE_VERSION (1); receiver rejects on mismatch |
| `reserved` | 1 B | MBZ |
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
| `mac` | 32 B | HMAC-SHA256 over (common header ‖ view ‖ seq ‖ digest ‖ payload). The common header is the FIRST 4 bytes — so `msg_type` (header byte 0) and `sender_id` (header byte 1) are BOTH bound by the MAC (anti-spoof). `msg_type` is NOT a standalone field between seq and digest. See §7 and [CRYPTO.md §5](./CRYPTO.md#5-per-message-mac). |

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
   Worst case (n_prepared=PBFT_VC_MAX_PREPARED=4): 88 + 4 × 40 = 248 bytes → fits ESP-NOW 250 B (H2 fix)
   Note: PBFT_VC_MAX_PREPARED is range-capped at 4 (Kconfig); if a future
   revision increases the cap, the MEMORY.md static_assert will fail at build time.
```

| Field | Size | Notes |
|-------|------|-------|
| `view` | 4 B | current view (suspected-faulty primary's view) |
| `new_view` | 4 B | next view to transition to |
| `checkpoint_seq` | 8 B | last stable checkpoint sequence number |
| `checkpoint_digest` | 32 B | digest of app state at checkpoint |
| `n_prepared` | 1 B | number of prepared entries (0..PBFT_VC_MAX_PREPARED, default 4) |
| prepared entries | 0–4000 B | each is `{seq:u64, digest:32}` = 40 B |
| `mac` | 32 B | HMAC over header + body (no MAC trailer on entries) |

**Kconfig:**

```kconfig
config PBFT_VC_MAX_PREPARED
    int "Max prepared entries carried in a View-Change"
    default 4
    range 0 4
    help
      Caps the worst-case View-Change size. With n_prepared=4 the
      worst-case VC = 248 B (fits ESP-NOW 250 B with 2 B spare);
      with 10 it would be 488 B (no longer fits). The default 4
      enforces the "no-runtime-fallback" rule: View-Change MUST
      always be ≤ 250 B at compile time. To allow more prepared
      entries you must also change the transport — see
      CONFIG_PBFT_TRANSPORT_ESP_NOW vs WIFI_UDP.
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

   Worst case (n=7, n_prepared=4 each VC): ≈ 1316 B
   + O-set (10 Pre-Prepares × 338 B): ≈ 3380 B
   Grand total worst case ≈ 4.7 KB → UDP only
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

For every message except the initial (TOFU) Hello, the MAC is
`HMAC-SHA256(hmac_keys[sender_id], mac_input)` producing a 32-byte tag, where:

```
mac_input = common_header ‖ view(u32 LE) ‖ sequence(u64 LE) ‖ digest[32] ‖ payload[payload_len]

common_header (4 bytes) = msg_type(u8) ‖ sender_id(u8) ‖ proto_version(u8) ‖ reserved(u8)
```

Key points (must match [CRYPTO.md §5](./CRYPTO.md#5-per-message-mac) exactly):
- `msg_type` is the **first byte of the common header**, NOT a standalone field
  placed between `sequence` and `digest`.
- `sender_id` (header byte 1) **is bound** by the MAC — this prevents a Byzantine
  node from replaying another node's message under a forged `sender_id`.
- Integers are little-endian; `digest`, `pubkey`, and the MAC tag are raw canonical bytes.

---

## 8. ESP-NOW size constraint (250 B) — implications

| Message | Size | Fits in ESP-NOW? |
|---------|------|------------------|
| Hello | 104 B (102 B wire + 2 B pad to align) | ✅ |
| Pre-Prepare (256 B payload) | 338 B | ❌ exceeds |
| Prepare | 80 B | ✅ |
| Commit | 80 B | ✅ |
| View-Change | 88 B | ✅ |
| New-View | ~2.5–6 KB (worst case) | ❌ exceeds |
| Checkpoint | 80 B | ✅ |

### 8.1 Solutions for oversized messages

> ⚠️ **Design constraint (user requirement, no runtime fallback):** esp-pbft does
> **NOT** fall back from ESP-NOW to UDP at runtime. The transport is a
> **compile-time** choice via `CONFIG_PBFT_TRANSPORT_ESP_NOW=y` or
> `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y` (mutually exclusive in a `choice` block).
> An oversized message over ESP-NOW returns `PBFT_NET_ERR_INVALID` at runtime
> and the caller must reconfigure at compile time if New-View or large
> Pre-Prepare payloads are required.

**Option A: Compile-time choice of Wi-Fi UDP** (recommended for New-View support)
- Set `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y` at compile time
- All messages (including New-View) use UDP multicast/unicast
- Selected when New-View (5.9 KB worst case) or Pre-Prepare with full
  `PBFT_TX_PAYLOAD_MAX = 256 B` payload (338 B) must be supported

**Option B: ESP-NOW only with payload size cap** (for minimal-RAM builds)
- Set `CONFIG_PBFT_TRANSPORT_ESP_NOW=y` and reduce `PBFT_TX_PAYLOAD_MAX` to ≤168 B
  so Pre-Prepare fits within 250 B
- **New-View is NOT supported in this configuration** — view-change is impossible;
  the cluster is permanently stuck on view 0 if the primary fails. Acceptable only
  for single-primary deployments where the primary cannot fail (e.g., AP node is
  externally powered and is the primary by design).
- A `_Static_assert` (see [MEMORY.md §X](./MEMORY.md)) checks at compile time:
  if ESP-NOW is selected and `PBFT_TX_PAYLOAD_MAX + 82 > 250`, build fails.

**Option C: Fragmentation** (rejected — see Open Questions P2)
- Split large messages across multiple ESP-NOW packets
- Adds reassembly code, sequence numbers, timeouts
- Not recommended for v1

**Decision:** **Option A or B at compile time** — no runtime fallback. The Kconfig
`choice` enforces mutual exclusion; `_Static_assert` enforces payload size at
compile time; `espnow_broadcast()` returns `PBFT_NET_ERR_INVALID` at runtime if a
malicious or buggy caller violates the compile-time constraint.

### 8.2 Single-transport runtime (compile-time-selected)

There is exactly **one** transport, chosen at compile time and exposed as
`pbft_default_transport` (= `pbft_get_transport()`, §2.1). Both Pre-Prepare and
New-View are sent through that same single transport — there is no separate
runtime `udp_transport`, and transports are **not** "both initialized at boot."

```c
esp_err_t pbft_send_pre_prepare(const pbft_msg_t* msg) {
    size_t total_len = msg->payload_len + 82;  // 82 = header overhead
    // No runtime fallback. The chosen transport was decided at compile time.
    // If ESP-NOW is selected and total_len > 250, this returns PBFT_NET_ERR_INVALID
    // (defensive runtime check; should be unreachable if _Static_assert passed).
    return pbft_default_transport->broadcast(msg, total_len);
}

esp_err_t pbft_send_new_view(const pbft_new_view_t* nv, size_t nv_len) {
    // Sent via the SAME single compiled default_transport. New-View (~5.9 KB worst
    // case) only fits when the build selected CONFIG_PBFT_TRANSPORT_WIFI_UDP. Under
    // CONFIG_PBFT_TRANSPORT_ESP_NOW, view-change (hence New-View) is NOT supported
    // (see §8.1, Option B); the call returns PBFT_NET_ERR_INVALID at runtime.
    return pbft_default_transport->broadcast(nv, nv_len);
}
```

Because view-change requires sending the ~5.9 KB New-View message, **view-change as
a feature requires the `CONFIG_PBFT_TRANSPORT_WIFI_UDP=y` build** (see §8.1). The
ESP-NOW build does not support New-View. Only the one selected transport is
initialized at boot.

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
      → if proto_version != PBFT_WIRE_VERSION → drop (count as invalid)  // §4.1
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
| Max packet size (New-View) | ~2.5–6 KB (worst case, see §6.6) | exceeds ESP-NOW → must use UDP |

---

## 13. Open questions (protocol-level)

| # | Question | Status |
|---|----------|--------|
| P1 | Should we use Wi-Fi UDP by default given New-View constraint? | 🟡 Open |
| P2 | ESP-NOW fragmentation worth it? | 🟡 No (UDP fallback simpler) |
| P3 | Should we add message-type versioning? | ✅ Resolved — `proto_version` byte in common header (§4.1), `PBFT_WIRE_VERSION=1`; receiver rejects on mismatch, freezing the v1 wire format |
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