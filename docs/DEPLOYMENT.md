# esp-pbft — Deployment Guide

> **Status:** 🟡 Draft (v0.1)
> **Scope:** provisioning flow, NVS image baking, node_id assignment, OTA strategy, monitoring, Y-5 operational runbook, physical-security requirements.
> **Out of scope:** build → see ESP-IDF docs; production topology → application docs.

This document is the **canonical reference for deploying esp-pbft in production**. It addresses audit issues **C2** (NVS layout undefined) and **C7** (TOFU physical-security assumption not explicit).

---

## 1. Deployment overview

```
   ┌────────────────────────────────────────────────────────────────┐
   │  Step 1: Factory provisioning                                  │
   │  ─ Bake NVS image with node_id, cluster config, timeouts      │
   │  ─ Flash bootloader + partition table + application           │
   │  ─ Print QR code label with node_id + MAC                     │
   │  ─ (Optional) Inject ECDSA pubkey via secure element          │
   └────────────────────────────────────────────────────────────────┘
                              ↓
   ┌────────────────────────────────────────────────────────────────┐
   │  Step 2: Field deployment                                     │
   │  ─ Power on each node                                         │
   │  ─ Verify all 7 nodes are on the same Wi-Fi channel           │
   │  ─ (Optional) Visually verify the QR codes match the layout   │
   │  ─ Cluster self-organizes via Pattern Y handshake             │
   └────────────────────────────────────────────────────────────────┘
                              ↓
   ┌────────────────────────────────────────────────────────────────┐
   │  Step 3: Operational monitoring                               │
   │  ─ Per-node logs (heap, metrics)                              │
   │  ─ Cluster-wide view (consensus progress)                     │
   │  ─ Y-5 re-gen scheduling                                      │
   │  ─ OTA firmware updates                                       │
   └────────────────────────────────────────────────────────────────┘
```

---

## 2. Step 1 — Factory provisioning

### 2.1 NVS partition image baking

ESP-IDF's `nvs_partition_gen.py` tool generates a binary blob that can be flashed to the `nvs` partition. esp-pbft uses **three namespaces** (per MEMORY.md §4):

```
# tools/nvs_config.csv
key,type,encoding,value
pbft,namespace,,
pbft_cluster,namespace,,
pbft_timeouts,namespace,,

# pbft namespace
pbft,my_node_id,hex2bin,00            # overwritten per device at flash time
pbft,view,u32,00000000
pbft,next_seq,u64,0000000000000000
pbft,low_wm,u64,0000000000000000
pbft,high_wm,u64,0000000000000064
pbft,log_size_max,u16,0064
pbft,payload_max,u16,0100
pbft,transport,u8,00

# pbft_cluster namespace (MAC addresses for all 7 nodes)
pbft_cluster,cluster_size,u8,07
pbft_cluster,node_0_mac,hex2bin,AABBCCDDEE00
pbft_cluster,node_1_mac,hex2bin,AABBCCDDEE01
pbft_cluster,node_2_mac,hex2bin,AABBCCDDEE02
pbft_cluster,node_3_mac,hex2bin,AABBCCDDEE03
pbft_cluster,node_4_mac,hex2bin,AABBCCDDEE04
pbft_cluster,node_5_mac,hex2bin,AABBCCDDEE05
pbft_cluster,node_6_mac,hex2bin,AABBCCDDEE06
pbft_cluster,node_0_ip,u32,0xC0A80401    # 192.168.4.1 (only if Wi-Fi UDP)
... (etc.)

# pbft_timeouts namespace
pbft_timeouts,prep_to,u32,000003E8       # 1000
pbft_timeouts,commit_to,u32,000003E8     # 1000
pbft_timeouts,vc_to,u32,000007D0         # 2000
pbft_timeouts,ckpt_to,u32,000007D0       # 2000
```

Generate the binary:

```bash
python $IDF_PATH/components/nvs_flash/nvs_partition_gen/nvs_partition_gen.py generate \
    tools/nvs_config.csv \
    build/nvs.bin \
    0x6000    # partition size
```

Flash to the `nvs` partition at offset `0x9000` (typical ESP32-C3 partition table):

```bash
esptool.py --chip esp32c3 --port /dev/ttyUSB0 \
    write_flash 0x9000 build/nvs.bin
```

### 2.2 Per-device `my_node_id` override

The same NVS image is flashed to all 7 devices. Each device's `my_node_id` is then written separately:

```bash
# For node 3:
python $IDF_PATH/components/nvs_flash/nvs_partition_gen/nvs_partition_gen.py \
    set-value --partition build/nvs.bin \
    --type u8 --key my_node_id --value 0x03 \
    --namespace pbft
esptool.py --chip esp32c3 --port /dev/ttyUSB0 \
    write_flash 0x9000 build/nvs.bin
```

Or use the on-device command at first boot (via console REPL or physical button):

```c
// In app code, gated by CONFIG_PBFT_PROVISIONING_MODE
void app_main(void) {
    if (provisioning_button_pressed()) {
        uint8_t id = read_id_from_potentiometer();  // 0..6
        pbft_set_node_id(id);
        ESP_LOGW(TAG, "Provisioning: node_id=%d written to NVS", id);
        esp_restart();
    }
    // ... normal startup ...
}
```

### 2.3 MAC address source

ESP32-C3 MAC is factory-programmed (Espressif OUI `A0:85:E3` or similar). For esp-pbft:
- Use factory MAC for ESP-NOW (broadcast slot is `FF:FF:FF:FF:FF:FF`)
- Use `esp_netif` DHCP-assigned IP for Wi-Fi UDP

If the customer wants custom MACs (e.g., for regulatory or branding reasons), use `esp_base_mac_addr_set()` at boot.

### 2.4 Production firmware layout

```
0x0000   bootloader (bootloader.bin)
0x8000   partition table (partitions.bin)
0x9000   nvs (nvs.bin — contains pbft config + cluster config)
0xE000   phy_init
0x10000  ota_0 (factory app)
0x200000 ota_1 (OTA update slot)
0x300000 storage (app data, optional)
```

---

## 3. Step 2 — Field deployment

### 3.1 Physical-security requirement (audit C7 fix)

**Critical assumption:** during initial boot, **all 7 nodes must be in physical proximity and on the same RF channel**. This is required because Pattern Y uses TOFU (Trust On First Use) — the first Hello from each `node_id` is trusted. An attacker on the same channel could otherwise substitute a pubkey.

Mitigations:

1. **Document:** "Cluster must be deployed in a physically secured area (e.g., locked cabinet, secure room). RF snooping is considered equivalent to physical access."
2. **Bootstrap attestation (v2):** operator presses a button on all 7 nodes within 30 s; only after this window does the cluster accept Hellos.
3. **Channel lock:** all nodes hard-coded to a non-default channel (e.g., channel 13 — less crowded in some regions). Reduces accidental RF collisions but does not defend against a deliberate attacker.

For high-security deployments (e.g., industrial control), v2 should add manufacturer-injected RSA-2048 signatures on Hello (see FAILURE-MODES.md §5.1).

### 3.2 Power-on sequence

1. Apply power to all 7 nodes within 60 s of each other
2. Each node:
   - Boots in ~250 ms
   - Generates TRNG keypair (~5 ms)
   - Delays randomly 0-3 s (boot stagger)
   - Broadcasts Hello
   - Receives 6 peer Hellos
   - Runs ECDH with each
   - Stores 6 HMAC keys
   - Begins PBFT (~3.5 s total per node)
3. Last node becomes PBFT-ready at ~5-7 s after power-on
4. Primary (node 0) starts receiving submissions

### 3.3 Verification

Console output (via `idf.py monitor` or UART):

```
I (3500) pbft: TRNG keygen complete (5 ms)
I (4000) pbft: Boot stagger 1.7 s
I (5700) pbft: Hello broadcast sent
I (6200) pbft: Hello from node 1 received (first)
I (6300) pbft: Hello from node 2 received
...
I (8200) pbft: 6 peer Hellos received
I (8500) pbft: ECDH with 6 peers complete (45 ms)
I (8500) pbft: PBFT ready (state=NORMAL, view=0, primary=0)
```

If the line "PBFT ready" appears within 10 s, deployment succeeded.

### 3.4 Common deployment issues

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| "No peer Hellos within 5 s" | RF interference, wrong channel, out of range | check channel; reduce distance |
| "TOFU mismatch: cached X, received Y" | node restarted with different key (expected) OR attacker (investigate) | if planned restart: ignore; otherwise alarm |
| "Primary unresponsive" | node 0 dead OR view-change failed | check node 0 power; check cluster log |
| High view-change rate (> 1/min) | RF noisy, timeouts too tight | increase prepare_timeout_ms; check Wi-Fi channel |

---

## 4. Step 3 — Operational monitoring

### 4.1 Per-node logging

Each node should expose logs via one of:

- UART (115200 baud) — for bench testing
- Syslog over UDP — for production (uses lwIP, available in Wi-Fi UDP build)
- MQTT publish to a cloud broker — for remote clusters

`pbft_log_cb` (configurable in `pbft_config_t`) is the hook:

```c
static void my_log_cb(int level, const char* msg) {
    if (level >= ESP_LOG_WARN) {
        mqtt_publish("pbft/alarm", msg);
    }
}

pbft_config_t cfg = {
    .log_cb = my_log_cb,
    // ...
};
```

### 4.2 Key metrics to monitor

| Metric | Threshold for alarm |
|--------|---------------------|
| `heap_min_free_bytes` | < 16 KB |
| `mac_failures` (per minute) | > 10 |
| `view_changes_initiated` (per hour) | > 5 |
| `prepare_timeouts` (per minute) | > 2 |
| `y5_regen_failures` (cumulative) | > 0 |

### 4.3 Cluster-wide view

A simple Python script aggregates per-node logs:

```python
# tools/cluster_monitor.py
import socket, json

def on_log(node_id, msg):
    rec = json.loads(msg)
    if rec['metric'] == 'mac_failures' and rec['value'] > 10:
        alarm(f"Node {node_id}: {rec['value']} MAC failures in last minute")
    if rec['metric'] == 'heap_min' and rec['value'] < 16384:
        alarm(f"Node {node_id}: heap low ({rec['value']} bytes)")
```

Run as a systemd service on a monitoring host.

---

## 5. OTA updates

### 5.1 Strategy

esp-pbft supports ESP-IDF's standard OTA via two partitions (ota_0 + ota_1).

**Constraint:** never run `pbft_stop` during OTA download; OTA runs in a separate task.

```c
// In app code:
void start_ota(const char* url) {
    pbft_stop();   // pause PBFT — cluster continues without this node
    esp_https_ota(url);  // download + verify + switch boot partition
    esp_restart();        // reboot into new firmware
}
```

During OTA:
- This node is absent for 30-60 s
- Other 6 nodes have quorum (5 of 6 needed for Prepare); consensus continues
- After reboot, this node re-handshakes and catches up

### 5.2 Cluster-wide OTA

For all 7 nodes, stagger the OTA over time:

```
t=0:00   Start OTA on node 6
t=0:30   node 6 rebooted; cluster catches it up
t=1:00   Start OTA on node 5
t=1:30   node 5 rebooted; cluster catches it up
...
t=6:00   All 7 nodes updated
```

Each individual OTA has < 60 s downtime. The cluster never loses quorum (always ≥ 5 honest nodes running).

#### Wire-version compatibility (G3 / PROTOCOL §4.1)

All nodes in a cluster **must** run the same `PBFT_WIRE_VERSION` (the `proto_version` byte in
the common wire header). The wire format is frozen within a major version:

- A rolling OTA **must NOT** change the wire format within a major version. If a release bumps
  `PBFT_WIRE_VERSION`, it is **not** a rolling-OTA-compatible upgrade — the cluster cannot be
  partially upgraded across a version boundary.
- During the rolling window above, every running node (old and new firmware) carries the **same**
  `PBFT_WIRE_VERSION`, so all packets interoperate.
- A receiver rejects any packet whose `proto_version != PBFT_WIRE_VERSION` (counted as a
  dropped/invalid packet — see PROTOCOL §4.1). A mismatched-version node therefore cannot
  participate in consensus; it is effectively absent until it runs a compatible build.

To change `PBFT_WIRE_VERSION` (a breaking wire change), schedule cluster downtime and flash all
nodes to the new major version together, rather than a rolling OTA.

### 5.3 OTA verification

After OTA, each node should:
- Verify firmware signature (use ESP-IDF's `esp_app_verify()` with secure boot V2)
- Run `pbft_init()` smoke test before resuming PBFT participation
- If smoke fails, roll back to previous partition automatically (ESP-IDF does this if `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`)

---

## 6. Y-5 operational runbook

Every 24 hours (staggered by `node_id × 1 hour`), each node regenerates its ECDSA keypair and triggers soft re-handshake.

### 6.1 Normal re-gen

- Time: 02:00 for node 2, 03:00 for node 3, etc.
- Duration: ~100 ms (keygen + ECDH × 6)
- Visible: brief spike in current (TRNG peak ~120 mA for 5 ms)
- App impact: NONE — peer re-handshake is transparent

### 6.2 Detected re-gen failure

If a peer's re-handshake fails (e.g., peer crashed mid-re-handshake):

```
W (5000) pbft: Re-handshake timeout for node 3 (re-gen in progress?)
W (6000) pbft: Peer 3 unresponsive — waiting for next Hello
```

Operator action:
- Wait 5 minutes (peer may reboot)
- If still unresponsive: investigate power/network
- Cluster continues with 5/6 active (quorum maintained)

### 6.3 Cluster-wide Y-5

Y-5 happens at different times per node (staggered). Worst case: 7 Y-5 events in 6 hours. Each event is independent and short. No coordinated action needed.

### 6.4 Y-5 tuning

```kconfig
CONFIG_PBFT_Y5_REGEN_PERIOD_HOURS=24
CONFIG_PBFT_Y5_STAGGER_HOURS=1
CONFIG_PBFT_Y5_GRACE_MS=5000    # audit B11 — keep old keys during re-handshake
```

For higher-security deployments:

```kconfig
CONFIG_PBFT_Y5_REGEN_PERIOD_HOURS=1
```

(1-hour compromise window; higher CPU cost due to frequent re-handshakes.)

### 6.5 ESP-IDF power-save Kconfig (mandatory for ESP-NOW RX duty-cycling)

The following ESP-IDF Kconfig entries **must** be set in `sdkconfig.defaults` for esp-pbft's ESP-NOW RX power-saving (POWER.md §9) to work in the disconnected state. They are required by `esp_now_set_wake_window()` per the [Espressif API reference](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html#config-esp-now-power-saving-parameter).

```kconfig
# ESP-NOW RX power save — required by esp_now_set_wake_window() in disconnected state
CONFIG_ESP_WIFI_STA_DISCONNECTED_PM_ENABLE=y

# Master switch for ESP-NOW RX duty-cycling (see POWER.md §9)
CONFIG_ESPNOW_ENABLE_POWER_SAVE=y

# 25% duty cycle (50 ms window per 200 ms interval) — tunable
CONFIG_ESPNOW_WAKE_WINDOW=50
CONFIG_ESPNOW_WAKE_INTERVAL=200
```

The first entry is the IDF v6.0.1 API contract; without it, `esp_now_set_wake_window()` is documented to no-op when the STA is disconnected. Even though most targets default it to `y`, pin it explicitly in `sdkconfig.defaults` so CI configurations cannot regress it (notably C5/C61 where the default depends on `CONFIG_ESP_HOST_WIFI_ENABLED`).

### 6.6 Replace a permanently dead node

A node that is permanently lost (bricked factory partition, hardware failure, physical loss)
must be physically replaced and the replacement re-provisioned into the cluster. The cluster's
fault tolerance is `f = 2`, so it tolerates **at most 2 simultaneous permanent failures**
(`n = 7`, quorum `2f+1 = 5`). If a third node is also down, restore at least one to a working
state before attempting a replacement.

Runbook (perform within the physical-security window — see §3.1, FAILURE-MODES §5.1):

1. **Provision the replacement** with the dead node's `my_node_id` (NVS), per §2.2. The new
   board has a different factory MAC, so the node-MAC table must be updated on every peer.
2. **Re-provision the node-MAC table on all surviving peers.** Write the replacement's MAC into
   the `pbft_cluster` NVS namespace (`node_<id>_mac`) on **all** peers, replacing the dead
   node's old MAC. (ESP-NOW uses the MAC directly; a stale MAC means peers never deliver to the
   replacement.)
3. **Clear the cached pubkey for that `node_id` on all peers** so TOFU re-captures the new key.
   Peer pubkeys are held in RAM only (G10), so a reboot of each peer already clears its cache;
   the replacement's first Hello is then trusted as a fresh TOFU. If you cannot reboot a peer,
   it must expose an operator hook to evict the cached pubkey for that `node_id`.
4. **Reboot the cluster (or the affected peers) within the physical-security window** so that
   every peer re-runs TOFU and captures the replacement's pubkey on the same trusted channel.
   Because pubkeys are RAM-only, each boot re-runs the discovery/TOFU window — the
   physical-security assumption (§3.1) must hold for **every** boot, not just first-ever
   deployment.
5. **Verify** the replacement reaches `PBFT ready` and that all peers log a Hello + ECDH for the
   new node (see §3.3). Confirm no `PBFT_ALARM_TOFU_MISMATCH` remains outstanding for that
   `node_id`.

> The MAC table lives in NVS; **pubkeys do not** (G10 — pubkeys are RAM-only, re-captured via
> TOFU on each boot). There is no NVS-cached pubkey that would let a reboot skip re-TOFU.

---

## 7. Production topology examples

### 7.1 Single cluster, single room

```
   ┌──────────────────────────────────────┐
   │  Room A (secured)                    │
   │   ┌──┐ ┌──┐ ┌──┐ ┌──┐               │
   │   │N0│ │N1│ │N2│ │N3│  1-3 m apart  │
   │   └──┘ └──┘ └──┘ └──┘               │
   │   ┌──┐ ┌──┐ ┌──┐                   │
   │   │N4│ │N5│ │N6│                   │
   │   └──┘ └──┘ └──┘                   │
   └──────────────────────────────────────┘
```

Suitable for: industrial control, lab equipment.

### 7.2 Multi-cluster (separate rooms)

Two independent clusters (different physical sites) cannot directly communicate via PBFT (no bridging). Each cluster has its own primary rotation and view-change.

For cross-cluster sync: use a higher-level application (e.g., periodic state snapshots over a separate link).

### 7.3 Cluster + edge gateway

```
   Cluster (7 nodes)         Edge gateway (separate)
   ┌─────────────────┐       ┌──────────────────┐
   │ N0..N6          │       │ Aggregator       │
   │   ↕ ESP-NOW     │←──────│ (Linux/Yocto)    │
   │                 │  UDP  │                  │
   └─────────────────┘       └──────────────────┘
                                    ↓
                              Cloud (MQTT)
```

The edge gateway participates as a "client" (submits TXs to node 0, receives commits via callback). It is NOT a PBFT replica (does not vote).

---

## 8. Application examples

### 8.1 Counter (canonical example)

`example/counter/main/main.c`:

```c
#include <stdio.h>
#include "pbft.h"
#include "esp_log.h"

static const char* TAG = "counter";
static int g_counter = 0;
static pbft_state_digest_fn_t g_prev_digest_cb;

static void on_commit(const pbft_tx_t* tx, uint64_t seq, void* ctx) {
    int delta;
    if (tx->payload_len != sizeof(delta)) {
        ESP_LOGW(TAG, "Bad payload length %u", tx->payload_len);
        return;
    }
    memcpy(&delta, tx->payload, sizeof(delta));
    g_counter += delta;
    ESP_LOGI(TAG, "seq=%llu delta=%d counter=%d", seq, delta, g_counter);
}

static void state_digest(uint64_t seq, uint8_t out_digest[32]) {
    // SHA-256(g_counter || seq) — toy example
    // In production use a proper hash (SHA-256 from mbedtls)
    char buf[64];
    int n = snprintf(buf, sizeof(buf), "%d|%llu", g_counter, seq);
    // ... compute SHA-256 of buf into out_digest ...
}

void app_main(void) {
    // 1. Cluster config
    static const pbft_node_desc_t cluster[7] = {
        {.node_id = 0, .mac_addr = {0xAA,0xBB,0xCC,0xDD,0xEE,0x00}},
        {.node_id = 1, .mac_addr = {0xAA,0xBB,0xCC,0xDD,0xEE,0x01}},
        // ... 5 more ...
    };
    pbft_config_t cfg = {
        .my_node_id = 0,    // per-device override
        .cluster_size = 7,
        .cluster_members = cluster,
        .prepare_timeout_ms = 1000,
        .commit_timeout_ms = 1000,
        .viewchange_timeout_ms = 2000,
        .payload_max = 256,
        .state_digest_cb = state_digest,
        .log_cb = NULL,
    };

    ESP_ERROR_CHECK(pbft_init(&cfg));
    ESP_ERROR_CHECK(pbft_register_commit_cb(on_commit, NULL));
    ESP_ERROR_CHECK(pbft_start());

    // 2. Periodic increment (every 1 s)
    while (1) {
        if (pbft_get_primary_id() != 0) {
            // Not the primary — skip submission
            vTaskDelay(pdMS_TO_TICKS(1000));
            continue;
        }
        int delta = 1;
        pbft_tx_t tx = {
            .payload = (uint8_t*)&delta,
            .payload_len = sizeof(delta),
            .flags = PBFT_TX_FLAG_DEDUP,
        };
        if (pbft_submit(&tx) != ESP_OK) {
            ESP_LOGW(TAG, "submit failed");
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 8.2 IoT sensor telemetry

Same pattern, but payload is a serialized struct:

```c
typedef struct {
    uint32_t device_id;
    int16_t  temperature_centi_c;
    uint16_t humidity_centi_pct;
    uint32_t timestamp_s;
} sensor_reading_t;

static void on_commit(const pbft_tx_t* tx, uint64_t seq, void* ctx) {
    sensor_reading_t r;
    if (tx->payload_len != sizeof(r)) return;
    memcpy(&r, tx->payload, sizeof(r));
    upload_to_cloud(&r);  // forward to cloud
}
```

---

## 9. Monitoring alerts cheat-sheet

| Alert | Meaning | Action |
|-------|---------|--------|
| `PBFT_ALARM_NO_PEERS` | < 4 Hellos received | check RF; check peer power |
| `PBFT_ALARM_TOFU_MISMATCH` | peer pubkey changed unexpectedly | investigate — possible attack |
| `PBFT_ALARM_CHECKPOINT_CONFLICT` | two checkpoint digests at same seq | log + view-change |
| `PBFT_ALARM_VIEW_CHANGE_LOOP` | > 10 view-changes in 5 min | investigate primary stability |
| `PBFT_ALARM_HEAP_LOW` | heap_min < 16 KB | restart affected node; check leak |
| `PBFT_ALARM_Y5_FAIL` | Y-5 re-gen failed | check TRNG; restart node |

---

## 10. Open questions (deployment level)

| # | Question | Recommendation |
|---|----------|----------------|
| D1 | Should we ship a Docker image with all 7 nodes virtualised? | Out of scope for v1; useful for CI testing. |
| D2 | Cloud-managed fleet (over-the-air config push)? | Out of scope; use ESP RainMaker or similar separately. |
| D3 | Secure boot V2 for firmware verification? | Yes, recommended (`CONFIG_SECURE_BOOT_V2=y`). |
| D4 | Flash encryption for stored config? | Recommended (`CONFIG_SECURE_FLASH_ENC_ENABLED=y`). esp-pbft's NVS namespace does NOT contain keys (CRYPTO §11) so encryption is for cluster-config confidentiality. |

---

## 11. References

- [HANDOVER.md](./HANDOVER.md) — high-level design
- [API-REFERENCE.md](./API-REFERENCE.md) — `pbft_init`, `pbft_register_commit_cb`
- [MEMORY.md](./MEMORY.md) §4 — NVS layout
- [CRYPTO.md](./CRYPTO.md) §11 — key persistence (none, by design)
- [FAILURE-MODES.md](./FAILURE-MODES.md) — recovery procedures
- [POWER.md](./POWER.md) — power calculations
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — C2 (NVS), C7 (physical security)
- ESP-IDF NVS Partition Generator: `components/nvs_flash/nvs_partition_gen/`
- ESP-IDF OTA docs: `api-reference/system/ota.rst`

---

**End of DEPLOYMENT.md (v0.1 draft — addresses audit issues C2, C7; full provisioning + OTA + runbook)**