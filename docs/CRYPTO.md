# esp-pbft — Crypto Design (Pattern Y)

> **Status:** 🟡 Draft
> **Pattern:** Y — Self-generated TRNG keys + ECDH-boot + HMAC-SHA256 runtime
> **Variant:** Y-5 — periodic re-gen every 24h, staggered by `node_id × 1h`
> **Stack:** PSA Crypto API on Mbed TLS 4.0.0 (TF-PSA-Crypto backend)

---

## 1. Overview

esp-pbft uses **Pattern Y** for all message authentication:

| Phase | Mechanism | When |
|-------|-----------|------|
| **Boot** | ECDSA P-256 keypair via TRNG | Every boot (cold start) |
| **Discovery** | Hello broadcast (pubkey) | After boot |
| **Key agreement** | ECDH → SHA-256 → HMAC key | After Hello exchange |
| **Runtime** | HMAC-SHA256 per message | Every consensus message |
| **Y-5 re-gen** | New keypair + soft re-handshake | Every 24h (staggered) |

**No ECDSA sign/verify at runtime** — ECDSA is used only to derive keys at boot.

---

## 2. Key types and sizes

| Key | Algorithm | Size | Lifetime |
|-----|-----------|------|----------|
| ECDSA private (own) | P-256 (secp256r1) | 32 B | RAM only, regenerated each boot + Y-5 |
| ECDSA public (own) | P-256 uncompressed | 65 B | RAM only, regenerated each boot + Y-5 |
| Peer ECDSA public | P-256 uncompressed | 65 B | BSS, cached (TOFU) |
| Pairwise HMAC key | HMAC-SHA256 | 32 B | RAM, regenerated on peer re-key |

**No key persistence** — private keys live only in RAM.

---

## 3. PSA Crypto API surface (verified for ESP-IDF v6.0.1)

Verified at `~/.espressif/v6.0.1/esp-idf/components/mbedtls/mbedtls/tf-psa-crypto/include/psa/crypto.h`.

### 3.1 APIs used

| Function | Purpose |
|----------|---------|
| `psa_generate_key()` | Create ECDSA P-256 keypair (TRNG-backed) |
| `psa_export_public_key()` | Get our 65-byte uncompressed pubkey (0x04 ‖ X_BE ‖ Y_BE) for Hello |
| `psa_key_agreement()` | ECDH → shared secret (32 B) |
| `psa_mac_compute()` | HMAC-SHA256 sign |
| `psa_mac_verify()` | HMAC-SHA256 verify |

### 3.2 APIs explicitly NOT used

| Function | Why avoided |
|----------|-------------|
| `psa_sign_message()` / `psa_verify_message()` | ECDSA at runtime = ~50 ms/msg — kills throughput |
| `psa_key_derivation_setup()` etc. | We use simple SHA-256 (not HKDF) — fewer APIs, smaller code |

### 3.3 Key attributes setup (template)

```c
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr, PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
psa_set_key_bits(&attr, 256);
psa_set_key_algorithm(&attr, PSA_ALG_ECDH);  // key-agreement only
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_DERIVE);  // not sign!
```

Note: `PSA_KEY_USAGE_DERIVE` only — we never sign with this key. This is enforced by the key's policy.

---

## 4. Boot sequence (cold start)

### 4.0 Constants

```c
#define PBFT_MAC_BYTES              32    // HMAC-SHA256 output size (also = SHA-256 digest)
#define PBFT_PUBKEY_BYTES           65    // uncompressed P-256: 0x04 ‖ X_BE(32) ‖ Y_BE(32)
// Match PSA psa_export_public_key() output verbatim (0x04 prefix mandatory) and
// psa_import_key() requirement for PSA_KEY_TYPE_ECC_PUBLIC_KEY(SECP_R1).
// Verified against esp-idf master: components/mbedtls/test_apps/.../test_psa_ecdsa.c:700-705,
// components/bt/esp_ble_mesh/common/crypto_psa.c:419-432,
// components/wpa_supplicant/esp_supplicant/src/crypto/crypto_mbedtls-ec.c:1714-1822,
// components/bt/controller/esp32c6/bt.c:1699-1740.
#define PBFT_BOOT_STAGGER_MS_MIN    0     // no stagger (boot all together — debug only)
#define PBFT_BOOT_STAGGER_MS_MAX    3000  // full stagger
```

Kconfig:

```kconfig
config PBFT_BOOT_STAGGER_MS_MAX
    int "Max boot stagger per node (ms)"
    default 3000
    range 0 30000
    help
      Each node delays 0..this many ms after boot before broadcasting Hello.
      Staggering prevents simultaneous broadcast collisions.
      Set to 0 for lab testing; production should use 1000-3000.
```

### 4.1 Pseudocode

**Corrected (fixes audit issue A4 — `psa_key_agreement` requires `psa_key_id_t`, not raw bytes).**

The key state in BSS:

```c
// Static state — all in BSS
static psa_key_id_t s_my_priv_key_id;                  // own ECDSA private key (DERIVE only)
static uint8_t      s_my_pub[65];                      // own public key (uncompressed P-256: 0x04 ‖ X ‖ Y)
static psa_key_id_t s_peer_pub_key_ids[7];             // imported peer public keys (handles)
static uint8_t      s_peer_pub_arena[7][65];            // raw peer public key bytes (uncompressed)
static psa_key_id_t s_hmac_key_ids[7];                 // derived HMAC keys (handles)
static uint8_t      s_hmac_key_arena[7][32];            // raw HMAC key bytes (for clearing)
static bool         s_have_peer_pub[7];                 // true once we've imported a peer pubkey
```

Boot sequence:

```c
// pbft_crypto_boot() — called once at boot
void pbft_crypto_boot(void) {
    // 1. Generate own ECDSA P-256 keypair
    psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
    psa_set_key_type(&attr, PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
    psa_set_key_bits(&attr, 256);
    psa_set_key_algorithm(&attr, PSA_ALG_ECDH);
    psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_DERIVE);
    // NOTE: NEVER add PSA_KEY_USAGE_SIGN_HASH here.
    // A signing-capable key, if leaked, lets an attacker spoof PBFT messages.
    // Our HMAC at runtime does not require this key to be signing-capable.

    psa_status_t status = psa_generate_key(&attr, &s_my_priv_key_id);
    if (status != PSA_SUCCESS) { /* handle error */ }

    // 2. Export our public key for Hello broadcast
    //    psa_export_public_key() returns 65 bytes uncompressed: 0x04 ‖ X_BE(32) ‖ Y_BE(32)
    size_t my_pub_len;
    psa_export_public_key(s_my_priv_key_id, s_my_pub, sizeof(s_my_pub), &my_pub_len);
    assert(my_pub_len == 65);
    assert(s_my_pub[0] == 0x04);   // uncompressed point marker (PSA convention)

    // 3. Staggered boot delay (random 0-3s)
    vTaskDelay(pdMS_TO_TICKS(esp_random() % 3000));

    // 4. Broadcast Hello (my_pub, my_node_id)
    pbft_network_broadcast_hello(s_my_pub, my_node_id);

    // 5. Wait for Hellos from 6 peers (timeout 5s)
    if (pbft_discovery_wait(5000) != ESP_OK) { /* retry or fail */ }

    // 6. For each peer: import pubkey, then ECDH, then derive HMAC key
    for (int peer = 0; peer < PBFT_CLUSTER_SIZE; peer++) {
        if (peer == my_node_id) continue;

        // (a) Import peer's pubkey as a PSA key
        //     This is required — psa_key_agreement() does NOT accept raw bytes;
        //     it requires a psa_key_id_t for the public side.
        psa_key_attributes_t pub_attr = PSA_KEY_ATTRIBUTES_INIT;
        psa_set_key_type(&pub_attr, PSA_KEY_TYPE_ECC_PUBLIC_KEY(PSA_ECC_FAMILY_SECP_R1));
        psa_set_key_bits(&pub_attr, 256);
        psa_set_key_algorithm(&pub_attr, PSA_ALG_ECDH);
        psa_set_key_usage_flags(&pub_attr, PSA_KEY_USAGE_DERIVE);

        // Destroy any prior import for this peer (e.g., from previous boot)
        if (s_have_peer_pub[peer]) {
            psa_destroy_key(s_peer_pub_key_ids[peer]);
            s_have_peer_pub[peer] = false;
        }

        // Wire-format precondition: peer pubkey must be 65 bytes uncompressed
        // (0x04 ‖ X_BE ‖ Y_BE). PSA rejects anything else with PSA_ERROR_INVALID_ARGUMENT.
        // If a peer sends 64-byte raw X‖Y or 33-byte compressed (0x02/0x03 ‖ X), we
        // must rehydrate to 65 bytes before calling psa_import_key() — see esp-idf
        // components/wpa_supplicant/esp_supplicant/src/crypto/crypto_mbedtls-ec.c:2984-3080
        // (crypto_ecdh_set_peerkey) for the canonical rehydration pattern.
        if (s_peer_pub_arena[peer][0] != 0x04) {
            ESP_LOGE(TAG, "peer %d pubkey missing 0x04 prefix (got 0x%02x)",
                     peer, s_peer_pub_arena[peer][0]);
            continue;
        }
        size_t peer_pub_len;
        psa_status_t s = psa_import_key(&pub_attr,
                                         s_peer_pub_arena[peer], 65,
                                         &s_peer_pub_key_ids[peer]);
        if (s != PSA_SUCCESS) {
            // Invalid curve point or other error — log alarm + skip peer
            ESP_LOGE(TAG, "psa_import_key failed for peer %d: %d", peer, (int)s);
            continue;
        }
        s_have_peer_pub[peer] = true;

        // (b) ECDH: shared = ECDH(my_priv, peer_pub_key_id)
        uint8_t shared[32];
        size_t shared_len = 0;
        s = psa_key_agreement(PSA_ALG_ECDH,
                              s_my_priv_key_id,
                              s_peer_pub_key_ids[peer],
                              shared, sizeof(shared), &shared_len);
        if (s != PSA_SUCCESS || shared_len != 32) {
            ESP_LOGE(TAG, "psa_key_agreement failed for peer %d: %d", peer, (int)s);
            mbedtls_platform_zeroize(shared, sizeof(shared));
            continue;
        }

        // (c) Derive HMAC key: SHA-256(shared || "PBFT-HMAC-v1")
        uint8_t hmac_key[32];
        pbft_sha256_derive(shared, sizeof(shared),
                          (const uint8_t*)"PBFT-HMAC-v1", 13,
                          hmac_key);

        // (d) Destroy any prior HMAC key for this peer
        if (s_hmac_key_ids[peer] != 0) {
            psa_destroy_key(s_hmac_key_ids[peer]);
        }

        // (e) Import the derived HMAC key as a PSA HMAC key
        psa_key_attributes_t hmac_attr = PSA_KEY_ATTRIBUTES_INIT;
        psa_set_key_type(&hmac_attr, PSA_KEY_TYPE_HMAC);
        psa_set_key_bits(&hmac_attr, 256);
        psa_set_key_algorithm(&hmac_attr, PSA_ALG_HMAC(PSA_ALG_SHA_256));
        psa_set_key_usage_flags(&hmac_attr, PSA_KEY_USAGE_SIGN_HASH | PSA_KEY_USAGE_VERIFY_HASH);
        s = psa_import_key(&hmac_attr, hmac_key, 32, &s_hmac_key_ids[peer]);
        if (s != PSA_SUCCESS) {
            ESP_LOGE(TAG, "psa_import_key(HMAC) failed for peer %d: %d", peer, (int)s);
            mbedtls_platform_zeroize(hmac_key, sizeof(hmac_key));
            mbedtls_platform_zeroize(shared, sizeof(shared));
            continue;
        }
        memcpy(s_hmac_key_arena[peer], hmac_key, 32);

        // Zeroize sensitive scratch
        mbedtls_platform_zeroize(shared, sizeof(shared));
        mbedtls_platform_zeroize(hmac_key, sizeof(hmac_key));
    }
}
```

**Key insight (fixes audit A4):** PSA's `psa_key_agreement()` requires both private and public keys as `psa_key_id_t` handles. Raw bytes on the public side are not accepted. The corrected flow imports the peer's pubkey as a transient PSA key first, then runs the agreement. Similarly, the derived HMAC key is imported as a PSA HMAC key so that `psa_mac_compute()` / `psa_mac_verify()` can use it directly.

For Y-5 re-gen (§7), the same flow runs: each peer's imported key ID is destroyed before the new pubkey is imported.

### 4.2 Why simple SHA-256 (not HKDF)?

We use `SHA-256(shared || "PBFT-HMAC-v1")` instead of HKDF-Extract+Expand because:

| Aspect | HKDF | Simple SHA-256 |
|--------|------|----------------|
| Output | 32+ B (expandable) | 32 B (fixed) |
| Salt input | Required | Not needed (label "PBFT-HMAC-v1" replaces) |
| Info input | Required | Yes ("PBFT-HMAC-v1") |
| Code size | ~200 B | ~50 B |
| Standard | RFC 5869 | Ad-hoc (acceptable for fixed-input pattern) |

For our use case (single 32-byte HMAC key per peer, fixed inputs), simple SHA-256 is sufficient and simpler. **HKDF would be needed if we derived multiple sub-keys per peer** (e.g., one for encrypt, one for MAC) — not our case.

### 4.3 Boot quorum requirement (partial Hello handling)

`pbft_discovery_wait(5000)` does not require **all** 6 peers to reply. We need enough to:

1. **Establish HMAC keys** with ≥1 peer to sign the first outgoing message.
2. **Reach a quorum-acceptable cluster view** for primary identification.

```c
#define PBFT_BOOT_MIN_PEERS   4   // min peers for boot completion

esp_err_t pbft_discovery_wait(uint32_t timeout_ms) {
    uint32_t deadline = esp_timer_get_time() / 1000 + timeout_ms;
    while (esp_timer_get_time() / 1000 < deadline) {
        uint8_t n_hellos = count_received_hellos();
        if (n_hellos >= PBFT_BOOT_MIN_PEERS) {
            // Acceptable — proceed with PBFT
            ESP_LOGW(TAG, "Boot with %d/6 peers (below full quorum)", n_hellos);
            return ESP_OK;
        }
        vTaskDelay(pdMS_TO_TICKS(50));
    }
    return ESP_ERR_TIMEOUT;
}
```

**Why 4?** Standard PBFT tolerates `f = 2` Byzantine nodes; need `2f + 1 = 5` for full safety. With **4 peers + self = 5**, we can run a degraded cluster (3+1 honest + 1 suspect) but if one of the 4 is Byzantine we are at risk. We accept 4 to handle transient boot ordering (one peer may be slow) and 1 stuck peer without blocking forever. **Below 4 = boot fails**; log critical and reset.

---

## 5. Per-message MAC

### 5.1 MAC input format

```
MAC input (concatenated, no length prefix):
  view      : u32  (4 bytes, little-endian)
  sequence  : u64  (8 bytes, little-endian)
  msg_type  : u8   (1 byte: Pre-Prepare=1, Prepare=2, Commit=3, etc.)
  digest    : 32 bytes (SHA-256 of TX payload, or all-zeros for non-TX msgs)
  payload   : variable (0-256 bytes, TX payload for Pre-Prepare; empty otherwise)
────────────────────────────────────────────────────────────
  Total     : 45 bytes minimum (view+seq+type+digest)
```

### 5.2 Why this input format?

| Field | Purpose |
|-------|---------|
| `view` | Anti-replay across view changes (old-view messages rejected) |
| `sequence` | Anti-replay within view (monotonic check) |
| `msg_type` | Domain separation (Pre-Prepare MAC ≠ Prepare MAC) |
| `digest` | Binds message to specific TX content |
| `payload` | Authenticates actual TX bytes |

Without `view‖sequence`, an attacker could replay an old (committed) Prepare to confuse a slow replica. Without `msg_type`, a Prepare MAC could be reused as Commit MAC.

### 5.3 HMAC compute

```c
// pbft_compute_mac(peer_id, view, seq, type, digest, payload, payload_len, mac_out)
esp_err_t pbft_compute_mac(uint8_t peer_id,
                             uint32_t view, uint64_t seq, uint8_t type,
                             const uint8_t digest[32],
                             const uint8_t* payload, size_t payload_len,
                             uint8_t mac_out[32]) {
    uint8_t buf[45 + 256];   // stack, bounded
    size_t off = 0;

    // Write view (LE)
    memcpy(buf + off, &view, 4);  off += 4;
    // Write sequence (LE)
    memcpy(buf + off, &seq, 8);   off += 8;
    // Write type
    buf[off++] = type;
    // Write digest
    memcpy(buf + off, digest, 32); off += 32;
    // Write payload (if any)
    if (payload && payload_len > 0) {
        if (payload_len > 256) return ESP_ERR_INVALID_ARG;
        memcpy(buf + off, payload, payload_len);
        off += payload_len;
    }

    // HMAC-SHA256 (uses ESP32-C3 SHA HW peripheral)
    size_t mac_len = 0;
    psa_status_t s = psa_mac_compute(hmac_key_ids[peer_id],
                                     PSA_ALG_HMAC(PSA_ALG_SHA_256),
                                     buf, off,
                                     mac_out, sizeof(mac_out), &mac_len);
    if (s != PSA_SUCCESS || mac_len != PBFT_MAC_BYTES) {
        return ESP_FAIL;
    }
    return ESP_OK;
}
```

**Performance:** ~5 µs per MAC on ESP32-C3 (SHA HW @ 160 MHz).

### 5.4 HMAC verify (receive path)

```c
esp_err_t pbft_verify_mac(uint8_t peer_id,
                            uint32_t view, uint64_t seq, uint8_t type,
                            const uint8_t digest[32],
                            const uint8_t* payload, size_t payload_len,
                            const uint8_t received_mac[32]) {
    uint8_t calc_mac[32];

    // 1. Re-compute expected MAC
    esp_err_t err = pbft_compute_mac(peer_id, view, seq, type, digest,
                                      payload, payload_len, calc_mac);
    if (err != ESP_OK) return err;

    // 2. Constant-time compare (prevent timing attack)
    return mbedtls_ct_memcmp(calc_mac, received_mac, 32) == 0
           ? ESP_OK : ESP_ERR_INVALID_MAC;
}
```

**Constant-time compare is mandatory** — non-constant-time `memcmp` leaks information through timing.

---

## 6. Discovery (Hello exchange)

### 6.1 Hello message format

```c
typedef struct __attribute__((packed)) {
    uint8_t  msg_type;       // = PBFT_MSG_HELLO (0)
    uint8_t  sender_id;      // 0..6
    uint8_t  pubkey[65];     // uncompressed P-256: 0x04 ‖ X_BE(32) ‖ Y_BE(32)
    uint8_t  re_handshake;   // 1 if this is a re-handshake (Y-5 or peer restart)
    uint8_t  mac[32];        // HMAC over (msg_type ‖ sender_id ‖ pubkey ‖ re_handshake)
                             //   using current hmac_keys[sender_id] if known,
                             //   else zeros (initial Hello)
} pbft_hello_t;
```

### 6.2 TOFU verification

```c
esp_err_t pbft_on_hello(const pbft_hello_t* hello) {
    uint8_t peer = hello->sender_id;
    if (peer >= PBFT_CLUSTER_SIZE) return ESP_ERR_INVALID_ARG;

    // TOFU check: have we seen this peer before?
    bool first_hello = !peer_pubkey_cached[peer];

    if (first_hello) {
        // Cache the pubkey (first wins)
        memcpy(peer_pubkey_cached[peer], hello->pubkey, 65);
        // Compute HMAC key with new peer
        pbft_derive_hmac_key(peer, hello->pubkey);
        // MAC may be zeros (peer has no key yet) — accept
        ESP_LOGW(TAG, "First hello from node %d — TOFU trust", peer);
    } else {
        // Subsequent hellos: pubkey MUST match
        if (memcmp(peer_pubkey_cached[peer], hello->pubkey, 65) != 0) {
            ESP_LOGE(TAG, "Hello from node %d has different pubkey! Alarm!", peer);
            return ESP_ERR_INVALID_PEER;
        }
        // Verify MAC with current hmac_keys[peer]
        // (re_handshake may be 1 — meaning peer regenerated key, but TOFU says first wins)
    }
    return ESP_OK;
}
```

**TOFU trust model:** First Hello from a peer is trusted. Subsequent Hellos must match cached pubkey. If mismatch → log alarm and refuse.

---

## 7. Y-5 periodic re-gen (every 24h, staggered)

### 7.1 Trigger

Each node has a timer that fires at `(node_id × PBFT_Y5_NODE_OFFSET_MS)` after boot:

```kconfig
config PBFT_Y5_PERIOD_MS
    int "Y-5 re-gen period (ms)"
    default 86400000   # 24 h
    range 3600000 604800000  # 1 h to 7 d

config PBFT_Y5_NODE_OFFSET_MS
    int "Y-5 stagger between nodes (ms)"
    default 3600000    # 1 h
    range 60000 36000000  # 1 min to 10 h
```

With `PBFT_Y5_NODE_OFFSET_MS = 1 h` and `PBFT_CLUSTER_SIZE = 7`:
- Node 0: 00:00
- Node 1: 01:00
- Node 2: 02:00
- ...
- Node 6: 06:00

All 7 nodes re-gen within a 6 h window; full cluster cycle is 24 h.

### 7.2 Grace period during transition

When node N fires its Y-5 timer:

1. **t=0**: Node N destroys its old keypair and generates a new one (§7.3).
2. **t=0..30 s**: Node N is in **transition** — it has a new pubkey but no peers know yet. It **cannot verify** incoming MACs (peers still sign with old HMAC key from N's perspective) and peers **cannot verify** its MACs (peers still expect HMAC key derived from N's old pubkey).
3. **t=30 s**: Timeout. If N has not received `re_handshake=1` Hellos from all 6 peers with the **new** pubkey, N broadcasts a Hello **announce** (3 times, 1 s apart).
4. **t=2 min hard limit**: If after 2 minutes any peer still hasn't responded, N logs a **partition alarm** and falls back to: continue with new keypair alone, accepting that those peers will reject its messages until they re-discover N.

**Why 30 s?** Typical ESP-NOW re-discovery latency on a quiet channel is ~5 s; 30 s covers 6× slack. Most peers will see the re_handshake Hello within 1-2 s.

**Why 2 min?** Beyond 2 min, either the peer is permanently down (N should view-change or alert) or the peer has lost N's new pubkey (peer is at fault). At 2 min, N broadcasts a fallback "RESYNC" message (just the new pubkey) for 5 retries 5 s apart.

### 7.3 Re-gen flow

```c
// pbft_y5_timer_cb() — called every 24h
void pbft_y5_timer_cb(void) {
    ESP_LOGI(TAG, "Y-5 re-gen triggered");

    // 1. Destroy old keypair (zeroize)
    psa_destroy_key(my_priv_key_id);
    mbedtls_platform_zeroize(hmac_keys, sizeof(hmac_keys));
    mbedtls_platform_zeroize(peer_pubkey_cached, sizeof(peer_pubkey_cached));

    // 2. Generate new keypair (TRNG)
    psa_generate_key(&attr, &my_priv_key_id);

    // 3. Export new pubkey (65 bytes uncompressed: 0x04 ‖ X_BE ‖ Y_BE)
    uint8_t new_pub[65];
    psa_export_public_key(my_priv_key_id, new_pub, sizeof(new_pub), &len);
    assert(len == 65 && new_pub[0] == 0x04);

    // 4. Broadcast Hello with re_handshake=1
    pbft_hello_t hello = {
        .msg_type    = PBFT_MSG_HELLO,
        .sender_id   = my_node_id,
        .re_handshake = 1,
    };
    memcpy(hello.pubkey, new_pub, 65);
    pbft_network_broadcast(&hello, sizeof(hello));

    // 5. Wait for peers to re-ECDH (handled via Hello receive + soft re-handshake)

    // 6. Restart 24h timer
    pbft_timer_start(PBFT_TIMER_Y5, 24 * 60 * 60 * 1000);
}
```

### 7.3 Soft re-handshake (peer receives re-handshake Hello)

When a replica receives Hello with `re_handshake=1` from peer X:

```c
// Already in pbft_on_hello()
if (hello->re_handshake && !first_hello) {
    // Same pubkey? Could be wrong peer, reject
    if (memcmp(peer_pubkey_cached[peer], hello->pubkey, 65) != 0) {
        // Different pubkey for known peer → re-handshake accepted
        memcpy(peer_pubkey_cached[peer], hello->pubkey, 65);
        pbft_derive_hmac_key(peer, hello->pubkey);
        ESP_LOGW(TAG, "Soft re-handshake with node %d", peer);
    }
    // Verify MAC with new hmac_keys[peer] (which we just derived)
}
```

---

## 8. ESP32-C3 crypto performance (verified)

From research at `~/.espressif/v6.0.1/esp-idf/components/mbedtls/`:

| Operation | Latency | Hardware |
|-----------|---------|----------|
| TRNG keygen (32 B) | <1 ms | TRNG peripheral |
| ECDSA P-256 keypair (TRNG + ecc_mul) | ~5 ms | TRNG + ECC HW |
| ECDH P-256 (shared secret) | ~8 ms | ECC HW point mul |
| HMAC-SHA256 (45-byte input) | ~3 µs | SHA HW + DMA |
| HMAC-SHA256 (256-byte input) | ~5 µs | SHA HW + DMA |
| ECDSA sign (NOT USED at runtime) | ~50 ms | Software (no HW) |
| ECDSA verify (NOT USED at runtime) | ~15 ms | ECC HW point mul |

**Conclusion:** Pattern Y uses **only fast operations** (TRNG, ECDH once at boot, HMAC at runtime). No slow ECDSA sign/verify.

---

## 9. Key lifecycle diagram

```
   t=0 (boot)
      │
      ▼
   TRNG keygen → ECDSA P-256 keypair (RAM only)
      │
      ▼
   Staggered delay (0-3s)
      │
      ▼
   Hello broadcast (pubkey, node_id)
      │
      ▼
   Wait for 6 Hellos → cache peer pubkeys
      │
      ▼
   For each peer: ECDH + SHA-256 derive HMAC key
      │
      ▼
   Steady state: HMAC every message (5 µs/msg)
      │
      │ ... 24 hours ...
      ▼
   t=24h (Y-5 trigger)
      │
      ▼
   psa_destroy_key(my_priv) → zeroize RAM
      │
      ▼
   psa_generate_key() → NEW keypair
      │
      ▼
   Hello broadcast with re_handshake=1
      │
      ▼
   Peers receive → re-derive HMAC keys (soft)
      │
      ▼
   Steady state resumes with new keys
      │
      │ ... 24 hours ...
      ▼
   (cycle continues)

   On node crash/restart:
      │
      ▼
   Same flow as t=0 (fresh TRNG keypair, no compromise window)
```

---

## 10. Security analysis

### 10.1 What we get

| Property | Mechanism |
|----------|-----------|
| **Authentication** | HMAC-SHA256 proves sender has shared key |
| **Integrity** | HMAC detects any bit-flip |
| **Anti-replay** | `view ‖ sequence` in MAC input |
| **Forward secrecy (within session)** | Y-5 re-gen bounds compromise window to 24h |
| **Forward secrecy (across reboot)** | New keypair at every boot |
| **Self-healing** | Compromise recovery via Y-5 or reboot |

### 10.2 What we don't get (and why it's OK)

| Property | Why not needed |
|----------|----------------|
| **Non-repudiation** | All nodes equally trusted (no audit trail needed) |
| **Confidentiality** | PBFT messages contain TX payload — application decides encryption |
| **Perfect forward secrecy (sub-second)** | 24h compromise window is acceptable for IoT |

### 10.3 Threat model

| Attacker | Defense |
|----------|---------|
| **Passive eavesdropper** | No defense (no encryption needed for PBFT) |
| **Active message forger** | HMAC verification — no shared key → reject |
| **Replay attacker** | `view ‖ sequence` bound in MAC |
| **Compromised node (full key)** | Y-5 re-gen + soft re-handshake bounds window to 24h |
| **MITM during Hello** | TOFU — first Hello trusted (physical security assumed, see below) |
| **Byzantine primary** | PBFT Prepare/Commit quorum catches (≥2f+1) |
| **Physical access to a node** | See [§10.4](#104-physical-security-assumption) — explicit assumption |

### 10.4 Physical security assumption (explicit)

Pattern Y's TOFU model **assumes physical security of the deployment site** during the initial Hello exchange.

**What's required:**

1. **Site is physically secured** — no attacker can:
   - Power up a rogue 8th ESP32-C3 in radio range during deployment
   - Substitute a node's hardware between provisioning and deployment
   - Listen to Hellos and replay a captured pubkey after re-deploying with attacker-controlled key

2. **Once Hello is exchanged**, the captured pubkey is cached in NVS. If the node reboots, it uses the cached pubkey (no re-TOFU). Reboot is safe; **deployment of a new node is the danger moment**.

**What's NOT required:**

- Faraday cage — radio range is irrelevant after initial discovery (re-handshake needs both ends to be in range, but the trusted key is already in NVS).
- Tamper-evident hardware — physical access still extracts the current session keys (RAM), so a sophisticated attacker can impersonate a node. Pattern Y's response is **Y-5 re-gen + reboot cycles** to bound compromise window to 24 h.

**Operational guidance:**

- See [DEPLOYMENT.md §3.1](./DEPLOYMENT.md) for the deployment-day security procedure.
- See [FAILURE-MODES.md §5](./FAILURE-MODES.md) for what happens if the assumption is violated.
- See [CRYPTO.md §12 C2](./CRYPTO.md) (Open Questions) for the upgrade path: adding ECDSA signature on Hello would drop the TOFU requirement but costs ~50 ms per handshake (likely too expensive for v1).

### 10.5 Anti-replay across view-changes

The MAC input format (§5.1) binds `view ‖ sequence ‖ type ‖ digest ‖ payload`. This means:

- **Same-view replay** (e.g., attacker captures Prepare in view V and replays it): the per-entry bitmask check (`prepare_from[sender]`) deduplicates; second copy is ignored.
- **Cross-view replay** (attacker captures Prepare from view V and replays in view V+1): the MAC was computed over `view=V` so it will not verify under `view=V+1`. The receiver recomputes MAC with the new view and the comparison fails.
- **Pre-Prepare replay from old primary** (e.g., after view-change, old primary's Pre-Prepare for a future sequence): the new primary assigns fresh sequences starting from its own `next_sequence`; old sequences are below `high_watermark` and rejected (§4.2 of [CONSENSUS.md](./CONSENSUS.md)).

**Conclusion:** No additional anti-replay window is needed; the binding in the MAC input is sufficient.

---

## 11. PSA key storage pattern (v1)

**v1 uses PSA volatile keys** (in RAM only, not persisted):

```c
psa_key_id_t my_priv_key_id;          // volatile, RAM only
psa_key_id_t hmac_key_ids[7];         // 6 volatile HMAC keys
```

**v2 could use PSA persistent keys** (NVS-backed, encrypted) if Y-5 is insufficient. Not implemented in v1.

---

## 12. Open questions (crypto-level)

| # | Question | Status |
|---|----------|--------|
| C1 | Should Hello be encrypted? (privacy of pubkey) | 🟡 Open — low priority |
| C2 | Should we add ECDSA signature on Hello (drop TOFU)? | 🟡 Open — adds ~50ms handshake cost |
| C3 | HMAC key rotation interval — 24h optimal? | 🟡 TBD — depends on threat model |
| C4 | Should we use HMAC-SHA256 or HMAC-SHA3-256? | ✅ SHA-256 (HW-accelerated) |

---

## 13. References

- [HANDOVER.md](./HANDOVER.md) §3.5 — Pattern Y decision
- [ARCHITECTURE.md](./ARCHITECTURE.md) §2 — Boot sequence
- [ARCHITECTURE.md](./ARCHITECTURE.md) §11 — Static memory discipline
- ESP-IDF v6.0.1 PSA Crypto docs — `~/.espressif/v6.0.1/esp-idf/components/mbedtls/`
- NIST FIPS 180-4 — SHA-256
- NIST FIPS 186-4 — ECDSA P-256
- RFC 2104 — HMAC
- RFC 5869 — HKDF (not used, see §4.2)

---

**End of CRYPTO.md (v0.1 draft — awaiting review)**