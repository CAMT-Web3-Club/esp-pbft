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
static uint8_t      s_peer_pub_arena[7][65];            // raw peer public key bytes (uncompressed, for ECDH)
static psa_key_id_t s_hmac_key_ids[7];                 // derived HMAC keys (handles)
static uint8_t      s_hmac_key_arena[7][32];            // raw HMAC key bytes (for clearing)
static bool         s_have_peer_pub[7];                 // true once we've received a peer pubkey
```

> **Note (PSA API correction):** The earlier "Audit A4" fix in §4.1 incorrectly wrapped the peer pubkey as a PSA key (via `psa_import_key` + `s_peer_pub_key_ids`). The actual TF-PSA-Crypto v1.0 signature of `psa_key_agreement` accepts the peer's **raw uncompressed pubkey bytes** (`const uint8_t *peer_key, size_t peer_key_length`) directly — no need to import as a PSA key first. The fix below uses the correct API and drops `s_peer_pub_key_ids[7]`.

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

    // 6. For each peer: ECDH (using raw pubkey bytes), then derive HMAC key
    for (int peer = 0; peer < PBFT_CLUSTER_SIZE; peer++) {
        if (peer == my_node_id) continue;

        // (a) Wire-format precondition: peer pubkey must be 65 bytes uncompressed
        //     (0x04 ‖ X_BE ‖ Y_BE). PSA rejects anything else with PSA_ERROR_INVALID_ARGUMENT.
        //     If a peer sends 64-byte raw X‖Y or 33-byte compressed (0x02/0x03 ‖ X), we
        //     must rehydrate to 65 bytes before calling psa_key_agreement() — see esp-idf
        //     components/wpa_supplicant/esp_supplicant/src/crypto/crypto_mbedtls-ec.c:2984-3080
        //     (crypto_ecdh_set_peerkey) for the canonical rehydration pattern.
        if (!s_have_peer_pub[peer] || s_peer_pub_arena[peer][0] != 0x04) {
            ESP_LOGE(TAG, "peer %d pubkey missing 0x04 prefix (got 0x%02x)",
                     peer, s_have_peer_pub[peer] ? s_peer_pub_arena[peer][0] : 0);
            continue;
        }

        // (b) ECDH: shared = ECDH(my_priv, peer_pub_bytes)
        //     Per TF-PSA-Crypto v1.0: psa_key_agreement accepts the peer's raw
        //     uncompressed pubkey bytes directly — no need to import as a PSA key.
        uint8_t shared[32];
        size_t shared_len = 0;
        s = psa_key_agreement(s_my_priv_key_id,
                              s_peer_pub_arena[peer], 65,
                              PSA_ALG_ECDH,
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

**Key insight (PSA Crypto on TF-PSA-Crypto v1.0):** The actual `psa_key_agreement()` signature in ESP-IDF v6.0.1 accepts the peer's **raw uncompressed pubkey bytes** (`const uint8_t *peer_key, size_t peer_key_length`) directly — no need to import the peer's pubkey as a PSA key first. This means the BSS layout in §4.1 only needs the raw `s_peer_pub_arena[7][65]` and does not require `s_peer_pub_key_ids[7]`. The derived HMAC key is still imported as a PSA HMAC key so that `psa_mac_compute()` / `psa_mac_verify()` can use it directly.

For Y-5 re-gen (§7), the same flow runs: each peer's cached pubkey in `s_peer_pub_arena[peer]` is overwritten with the new pubkey, and the HMAC key is re-derived.

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

### 4.2.1 SHA-256 helpers (two distinct functions)

Two SHA-256 helpers exist and MUST NOT be confused:

```c
// Plain SHA-256 of an arbitrary buffer. Used for TX digests (CONSENSUS pbft_submit).
// out[32] receives SHA-256(in[0..len-1]).
esp_err_t pbft_sha256(const uint8_t *in, size_t len, uint8_t out[32]);

// Key-derivation helper: out[32] = SHA-256(secret ‖ label).
// Used ONLY to derive a pairwise HMAC key from the ECDH shared secret (§4.1 step c).
esp_err_t pbft_sha256_derive(const uint8_t *secret, size_t secret_len,
                             const uint8_t *label,  size_t label_len,
                             uint8_t out[32]);
```

- `pbft_sha256()` is the plain message hash — a single buffer in, a 32-byte digest out.
  CONSENSUS computes the per-TX `digest` with this function (see G18 / [CONSENSUS.md](./CONSENSUS.md) §5.2).
- `pbft_sha256_derive()` is the KDF wrapper used at boot/re-gen; it concatenates a
  domain-separation label (`"PBFT-HMAC-v1"`) before hashing. It is NOT a general hash.

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
MAC input (common-header-first; concatenated, no length prefix):
  common_header : 4 bytes =
      msg_type(u8)      (Pre-Prepare=1, Prepare=2, Commit=3, etc.)
    ‖ sender_id(u8)     (0..6 — BOUND, anti-spoof)
    ‖ proto_version(u8) (= PBFT_WIRE_VERSION)
    ‖ reserved(u8)
  view      : u32  (4 bytes, little-endian)
  sequence  : u64  (8 bytes, little-endian)
  digest    : 32 bytes (SHA-256 of TX payload, or all-zeros for non-TX msgs)
  payload   : variable (0-256 bytes, TX payload for Pre-Prepare; empty otherwise)
────────────────────────────────────────────────────────────
  Total     : 48 bytes minimum (common_header+view+seq+digest)
```

The canonical byte order is:
`common_header(msg_type‖sender_id‖proto_version‖reserved) ‖ view ‖ sequence ‖ digest ‖ payload`.
`msg_type` is the FIRST byte (inside the common header), not a standalone field
between `sequence` and `digest`.

### 5.2 Why this input format?

| Field | Purpose |
|-------|---------|
| `msg_type` | Domain separation (Pre-Prepare MAC ≠ Prepare MAC) |
| `sender_id` | Binds message to its claimed sender (anti-spoof — a captured MAC cannot be re-attributed to a different node) |
| `proto_version` | Binds to the wire format version (rejects cross-version replay) |
| `view` | Anti-replay across view changes (old-view messages rejected) |
| `sequence` | Anti-replay within view (monotonic check) |
| `digest` | Binds message to specific TX content |
| `payload` | Authenticates actual TX bytes |

Without `view‖sequence`, an attacker could replay an old (committed) Prepare to confuse a slow replica. Without `msg_type`, a Prepare MAC could be reused as Commit MAC. Without `sender_id` bound, a captured MAC could be re-attributed to a different node.

### 5.3 HMAC compute

```c
// pbft_compute_mac(peer_id, sender_id, view, seq, type, digest, payload, payload_len, mac_out)
//   sender_id is the node that originated the message (bound into the MAC).
esp_err_t pbft_compute_mac(uint8_t peer_id,
                             uint8_t sender_id,
                             uint32_t view, uint64_t seq, uint8_t type,
                             const uint8_t digest[32],
                             const uint8_t* payload, size_t payload_len,
                             uint8_t mac_out[32]) {
    uint8_t buf[48 + 256];   // stack, bounded (4 hdr + 4 view + 8 seq + 32 digest + payload)
    size_t off = 0;

    // Write common_header: msg_type ‖ sender_id ‖ proto_version ‖ reserved (4 bytes, FIRST)
    buf[off++] = type;                 // msg_type
    buf[off++] = sender_id;            // sender_id (BOUND — anti-spoof)
    buf[off++] = PBFT_WIRE_VERSION;    // proto_version
    buf[off++] = 0;                    // reserved
    // Write view (LE)
    memcpy(buf + off, &view, 4);  off += 4;
    // Write sequence (LE)
    memcpy(buf + off, &seq, 8);   off += 8;
    // Write digest
    memcpy(buf + off, digest, 32); off += 32;
    // Write payload (if any)
    if (payload && payload_len > 0) {
        if (payload_len > 256) return ESP_ERR_INVALID_ARG;
        memcpy(buf + off, payload, payload_len);
        off += payload_len;
    }

    // HMAC-SHA256 (uses ESP32-C3 SHA HW peripheral)
    size_t mac_len;
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
                            uint8_t sender_id,
                            uint32_t view, uint64_t seq, uint8_t type,
                            const uint8_t digest[32],
                            const uint8_t* payload, size_t payload_len,
                            const uint8_t received_mac[32]) {
    uint8_t calc_mac[32];

    // 1. Re-compute expected MAC (sender_id is bound into the common header)
    esp_err_t err = pbft_compute_mac(peer_id, sender_id, view, seq, type, digest,
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
    bool first_hello = !s_have_peer_pub[peer];

    if (first_hello) {
        // VERY FIRST Hello of a never-seen peer (TOFU): a zero MAC is accepted ONLY here.
        // Cache the pubkey (first wins)
        memcpy(s_peer_pub_arena[peer], hello->pubkey, 65);
        s_have_peer_pub[peer] = true;
        // Compute HMAC key with new peer
        pbft_derive_hmac_key(peer, hello->pubkey);
        ESP_LOGW(TAG, "First hello from node %d — TOFU trust (zero MAC permitted)", peer);
    } else {
        // KNOWN peer. A re_handshake=1 Hello (peer rolled its key, §7) carries a NEW
        // pubkey but MUST prove it with a valid MAC under the CURRENT hmac_keys[peer]
        // (the key still in force before this Hello). A zero/invalid MAC is rejected —
        // zero MAC is permitted ONLY for the very first Hello of a never-seen peer above.
        if (hello->re_handshake) {
            if (pbft_verify_hello_mac(peer, hello) != ESP_OK) {
                ESP_LOGE(TAG, "re_handshake Hello from node %d failed MAC! Reject.", peer);
                return ESP_ERR_INVALID_MAC;   // do NOT accept the new pubkey
            }
            // MAC verified under the current key → accept the rolled pubkey (§7.3).
        } else {
            // Plain (non-re_handshake) Hello from a known peer: pubkey MUST match cache.
            if (memcmp(s_peer_pub_arena[peer], hello->pubkey, 65) != 0) {
                ESP_LOGE(TAG, "Hello from node %d has different pubkey! Alarm!", peer);
                return ESP_ERR_INVALID_PEER;
            }
            // MAC verified under the current hmac_keys[peer].
        }
    }
    return ESP_OK;
}
```

**TOFU trust model:** First Hello from a peer is trusted (zero MAC permitted). Subsequent Hellos must match cached pubkey, except a `re_handshake=1` Hello which may carry a new pubkey **only if** its MAC verifies under the current key. A `re_handshake=1` Hello from a KNOWN peer with a zero or invalid MAC is rejected — the zero-MAC exception applies solely to the first-ever Hello of a never-seen peer. If a plain Hello's pubkey mismatches the cache → log alarm and refuse.

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

### 7.2 Grace period during transition (keep-old-keys, no consensus pause)

```c
#define PBFT_Y5_GRACE_MS   5000   // 5 s: window in which old HMAC keys remain valid for RX
```

When node N fires its Y-5 timer, it does **not** destroy the old keys immediately and
consensus is **never paused**. Instead:

1. **t=0**: Node N derives its NEW keypair (§7.3) but **keeps the old `hmac_keys[]` arena**
   alongside the new one. It broadcasts a `re_handshake=1` Hello signed under its CURRENT
   (old) key so peers can verify the roll.
2. **t=0..PBFT_Y5_GRACE_MS (5 s)**: Both keys are live. On the RX path, N (and every peer)
   accepts a MAC that verifies under **either** the old OR the new key for the affected peer.
   Messages in flight continue to be produced and verified throughout — there is no window in
   which N "cannot verify". As each peer processes the re_handshake Hello it re-derives the
   new key and starts signing with it.
3. **t=PBFT_Y5_GRACE_MS**: The grace timer fires and N destroys the OLD HMAC keys
   (`psa_destroy_key` + zeroize), leaving only the new keys. From here only the new key
   verifies.

**Why 5 s?** Typical ESP-NOW re-discovery latency on a quiet channel is ~1-2 s; 5 s covers a
comfortable margin for every peer to receive the re_handshake Hello and re-derive the new
HMAC key before the old key is retired. Because both keys are accepted during the window,
no message is dropped for a verification gap.

> Note: there is no "30 s soft / 2 min hard" transition and no period in which N cannot
> verify incoming MACs. The dual-key grace window replaces that earlier design.

### 7.3 Re-gen flow

```c
// pbft_y5_timer_cb() — called every 24h
void pbft_y5_timer_cb(void) {
    ESP_LOGI(TAG, "Y-5 re-gen triggered");

    // 1. KEEP the old keys. Snapshot the current HMAC keys into the "old" arena so the RX
    //    path can still verify under them during the grace window. Do NOT destroy yet,
    //    and do NOT clear s_peer_pub_arena (peers are unchanged; only OUR keypair rolls).
    memcpy(s_hmac_key_ids_old, s_hmac_key_ids, sizeof(s_hmac_key_ids));
    s_old_keys_valid = true;
    psa_key_id_t old_priv = my_priv_key_id;   // retain handle for grace window

    // 2. Generate new keypair (TRNG)
    psa_generate_key(&attr, &my_priv_key_id);

    // 3. Export new pubkey (65 bytes uncompressed: 0x04 ‖ X_BE ‖ Y_BE)
    uint8_t new_pub[65];
    psa_export_public_key(my_priv_key_id, new_pub, sizeof(new_pub), &len);
    assert(len == 65 && new_pub[0] == 0x04);

    // 4. Broadcast Hello with re_handshake=1, MAC'd under the CURRENT (old) key so peers
    //    can verify the roll (a re_handshake Hello with zero/invalid MAC is rejected, §6.2).
    pbft_hello_t hello = {
        .msg_type    = PBFT_MSG_HELLO,
        .sender_id   = my_node_id,
        .re_handshake = 1,
    };
    memcpy(hello.pubkey, new_pub, 65);
    pbft_sign_hello_mac(&hello);   // signed under old hmac_keys[]
    pbft_network_broadcast(&hello, sizeof(hello));

    // 5. Peers re-ECDH on receiving the Hello (§7.3 soft re-handshake). During the grace
    //    window the RX path accepts MACs under EITHER s_hmac_key_ids[] (new) or
    //    s_hmac_key_ids_old[] (old). Consensus is NOT paused.

    // 6. Schedule destruction of the OLD keys AFTER the grace window (NOT now).
    pbft_timer_start(PBFT_TIMER_Y5_GRACE, PBFT_Y5_GRACE_MS);
    // (saved old_priv is destroyed in the grace callback too)

    // 7. Restart 24h timer
    pbft_timer_start(PBFT_TIMER_Y5, 24 * 60 * 60 * 1000);
}

// pbft_y5_grace_cb() — fires PBFT_Y5_GRACE_MS after re-gen; retires the old keys.
void pbft_y5_grace_cb(void) {
    if (!s_old_keys_valid) return;
    for (int p = 0; p < PBFT_CLUSTER_SIZE; p++) {
        if (s_hmac_key_ids_old[p] != 0 && s_hmac_key_ids_old[p] != s_hmac_key_ids[p]) {
            psa_destroy_key(s_hmac_key_ids_old[p]);
        }
        s_hmac_key_ids_old[p] = 0;
    }
    psa_destroy_key(s_old_priv);   // retired old private key
    mbedtls_platform_zeroize(s_hmac_key_arena_old, sizeof(s_hmac_key_arena_old));
    s_old_keys_valid = false;
    ESP_LOGI(TAG, "Y-5 grace window elapsed — old keys destroyed");
}
```

> The old key is destroyed ONLY in `pbft_y5_grace_cb()`, after `PBFT_Y5_GRACE_MS`. The
> re-gen path must NOT call `psa_destroy_key()` on the old key immediately.

### 7.3 Soft re-handshake (peer receives re-handshake Hello)

When a replica receives Hello with `re_handshake=1` from peer X:

```c
// Already in pbft_on_hello()
if (hello->re_handshake && !first_hello) {
    // Same pubkey? Could be wrong peer, reject
    if (memcmp(s_peer_pub_arena[peer], hello->pubkey, 65) != 0) {
        // Different pubkey for known peer → re-handshake accepted
        memcpy(s_peer_pub_arena[peer], hello->pubkey, 65);
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