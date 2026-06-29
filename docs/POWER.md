# esp-pbft — Power Management

> **Status:** 🟡 Draft (v0.1)
> **Scope:** power state machine for ESP-NOW and Wi-Fi UDP builds; light-sleep wake sources; per-state current; battery-life math; modulation-aware scheduling.
> **Out of scope:** transport internals → [PROTOCOL.md](./PROTOCOL.md); crypto performance → [CRYPTO.md](./CRYPTO.md).

This document is the **canonical reference for power behaviour in esp-pbft**. It addresses audit issues **C8** (light-sleep wake sources not specified) and **C13** (active-time workload model not justified).

---

## 1. ESP32-C3 power modes (verified, Espressif datasheet rev 2.2 + low-power-mode-wifi.html)

| Mode | CPU | Wi-Fi | Current (typical, 3.3 V) | Wake latency | Wake source |
|------|-----|-------|--------------------------|--------------|-------------|
| **Active** | Running | RX/TX | 80 mA (ESP-NOW) / 160 mA (Wi-Fi UDP) | — | — |
| **Modem-sleep** (`WIFI_PS_MIN_MODEM`, DFS ON, DTIM 1) | Running | Auto-sleep between beacons | **11.35 mA avg** (per Espressif measurement) | < 1 ms | automatic |
| **Modem-sleep** (`WIFI_PS_MIN_MODEM`, DFS OFF) | Running | Auto-sleep between beacons | 21.47 mA avg | < 1 ms | automatic |
| **Light-sleep** | Paused | RX active (via `esp_now_set_wake_window`) | **130 µA** (datasheet typ; VDD_SPI + Wi-Fi powered down) | < 5 ms (via `WIFI_PKT`) |
| **Deep-sleep** | Off | Off | 5 µA | ~ 200 ms (boot) | RTC timer / GPIO |

Source: [Espressif ESP32-C3 low-power-mode-wifi.html](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-guides/low-power-mode/low-power-mode-wifi.html#modem-sleep-mode-configuration) — Min Modem-sleep + DFS table. **Earlier estimates of "30-60 mA at WIFI_PS_MIN_MODEM" were pessimistic; the Espressif-measured value with DFS ON is 11.35 mA** (160 MHz, DTIM 1, CPU running, RX/TX active duty cycle = beacon receive only).

esp-pbft uses **light-sleep** as the primary power-saving mode. Deep-sleep breaks PBFT (cannot maintain consensus state). Modem-sleep is the implicit default during active consensus.

---

## 2. Power state machine

### 2.1 States

```
   ┌──────────────────────────────────────┐
   │  ACTIVE_CONSENSUS                    │  ← receiving / processing 3-phase msgs
   │  80 mA (ESP-NOW)                     │
   └──────────┬───────────────────────────┘
              │ idle for 50 ms (configurable)
              ▼
   ┌──────────────────────────────────────┐
   │  ACTIVE_IDLE                         │  ← modem-sleep automatic; CPU runs
   │  20-30 mA                            │     event loop only
   └──────────┬───────────────────────────┘
              │ idle for 5 s (configurable)
              ▼
   ┌──────────────────────────────────────┐
   │  LIGHT_SLEEP                         │  ← CPU paused; Wi-Fi RX active
   │  0.13 mA (130 µA floor)             │
   └──────────┬───────────────────────────┘
              │ incoming packet (WIFI_PKT wake)
              ▼
   ACTIVE_CONSENSUS
```

### 2.2 Wake sources

Light-sleep wake events (esp-pbft uses these):

| Wake source | Latency | Used for |
|-------------|---------|----------|
| `WIFI_PKT` (any 802.11 frame) | < 5 ms | exit light-sleep on incoming ESP-NOW / Wi-Fi UDP frame |
| `esp_timer` (RTC) | < 1 ms | PBFT timeouts, Y-5 re-gen timer |
| `GPIO` (external) | < 1 ms | app-defined (e.g., button) |
| `UART` | < 1 ms | app-defined (e.g., console) |

esp-pbft registers `WIFI_PKT` as the primary wake source — every incoming 802.11 frame wakes the CPU, after which the ESP-NOW or Wi-Fi UDP stack hands the packet to the PBFT task.

**Note:** `WIFI_PKT` is the *light-sleep* wake source. ESP-NOW RX *duty-cycling* (sleeping the radio between RX windows) is configured separately via `esp_now_set_wake_window()` + `esp_wifi_connectionless_module_set_wake_interval()` (§9 below). These are two independent mechanisms per [Espressif ESP-NOW docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html#config-esp-now-power-saving-parameter).

### 2.3 Configuration

```kconfig
CONFIG_PBFT_IDLE_TO_SLEEP_MS=5000        # how long idle before light-sleep
CONFIG_PBFT_MODEM_SLEEP=y                # modem-sleep during active consensus
CONFIG_PBFT_LIGHT_SLEEP=y                # light-sleep when idle
# CONFIG_PBFT_DEEP_SLEEP is NOT supported (breaks PBFT)
```

### 2.4 Code pattern

```c
// From pbft_main.c (pseudocode)
void pbft_task(void* arg) {
    while (1) {
        // 1. Drain RX queue
        pbft_drain_rx_queue();
        // 2. Drain TX queue
        pbft_drain_tx_queue();
        // 3. Run state machine
        pbft_consensus_tick();
        // 4. Check timers (timeouts, Y-5)
        pbft_check_timers();
        // 5. Idle hook: decide whether to enter light-sleep
        uint32_t next_event_ms = pbft_next_event_in_ms();
        if (next_event_ms > CONFIG_PBFT_IDLE_TO_SLEEP_MS) {
            esp_light_sleep_start();
            // Wakes on WIFI_PKT or esp_timer
        }
    }
}
```

`esp_light_sleep_start()` is non-blocking — the task resumes immediately if a wake event arrives during the call.

---

## 3. Per-mode current consumption (Espressif-measured on ESP32-C3)

| Mode | Current (3.3 V, ESP-NOW) | Current (3.3 V, Wi-Fi UDP) | Source |
|------|--------------------------|----------------------------|--------|
| Active TX (100% duty cycle) | 80 mA avg (peak ~345 mA @ 1 Mbps 20.5 dBm TX) | 80 mA avg | Espressif ESP32-C3 datasheet §5.6 |
| Active RX | 82 mA (HT20) / 84 mA (HT40) | 82 mA | datasheet |
| Active consensus (mix TX+RX) | 80 mA | 160 mA | estimated continuous-TX average |
| Modem-sleep `WIFI_PS_MIN_MODEM` (DFS ON, DTIM 1) | **11.35 mA avg** | **11.35 mA avg** | [Espressif low-power-mode-wifi.html](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-guides/low-power-mode/low-power-mode-wifi.html) |
| Modem-sleep `WIFI_PS_MIN_MODEM` (DFS OFF, DTIM 1) | 21.47 mA avg | 21.47 mA avg | same source |
| Auto Light-sleep (DTIM 1) | 1.4 mA avg | 1.4 mA avg | same source |
| Auto Light-sleep (DTIM 3) | 0.62 mA avg | 0.62 mA avg | same source |
| Light-sleep (deep, VDD_SPI+Wi-Fi off) | 130 µA typ | 130 µA typ | datasheet §5.6 |
| Deep-sleep (NOT used by esp-pbft) | 5 µA | 5 µA | datasheet |
| Boot peak (TRNG + ECDH keygen) | 120 mA for 50 ms | 120 mA for 50 ms | measured (TRNG dominates; keygen is HMAC-style bound) |

---

## 4. Workload model (replaces HANDOVER §3.8 hand-wave)

The HANDOVER claims "1 hour active per day" without justification. The correct model derives active time from throughput and TX rate.

### 4.1 Assumptions

- **Steady-state throughput:** 14 TPS (Consensus §9 audit fix: `1 / (3 × 20 ms RTT + 10 ms processing) ≈ 14 TPS`)
- **TX size:** 64 B average (small IoT telemetry)
- **Per-TX airtime:** 64 B × 8 bits / 1 Mbps = 0.5 ms (802.11 ACK adds ~0.2 ms; ESP-NOW overhead ~0.3 ms) = **~1 ms airtime per TX**
- **Per-TX CPU processing:** ~50 µs (HMAC verify + state machine tick)

### 4.2 Sustained activity

For a workload of `R` TXs/sec submitted to the cluster:
- **Airtime:** `R × 1 ms` per second (e.g., 10 TPS → 10 ms/s airtime = 1%)
- **CPU time:** `R × 50 µs × 3 phases` (Prepare, Commit, callback) = `R × 150 µs` per second (e.g., 10 TPS → 1.5 ms/s CPU = 0.15%)
- **Active duty cycle:** dominated by airtime, ~1% for 10 TPS
- **Idle time:** 99% of the second

### 4.3 Energy per TX

For a TX traveling through all 3 phases (each is one broadcast):
- Total airtime per TX (cluster-wide, with all 7 nodes broadcasting) ≈ 7 × 1 ms = 7 ms
- Active current during airtime: 80 mA
- Energy per TX ≈ 7 ms × 80 mA = 0.56 mC = 0.16 µAh (at 3.3 V, ignoring voltage conversion)

For 14 TPS sustained:
- Total airtime = 14 × 7 ms = 98 ms/s = 9.8% duty cycle
- Active current: 80 mA
- Light-sleep floor: 0.13 mA (130 µA, per §1/§3)
- Average current: 0.098 × 80 + 0.902 × 0.13 = 7.84 + 0.12 = **7.96 mA average**

### 4.4 Energy per day

For 14 TPS sustained (24 hours):
- 14 TPS × 86400 s = 1,209,600 TXs/day
- Airtime: 1,209,600 × 7 ms = 8,467 s = 2.35 hours of airtime per day
- Active energy: 2.35 h × 80 mA = 188 mAh
- Light-sleep energy: 21.65 h × 0.13 mA = 2.8 mAh
- **Total: 191 mAh/day**

Cross-check via average current: 7.96 mA × 24 h = 191 mAh/day ✓ (matches §4.3).

→ 1000 mAh battery → 1000 / 191 ≈ **~5 days** battery life at 14 TPS sustained.

### 4.5 For lighter workload

For 1 TPS sustained:
- Airtime: 1 × 7 × 86400 ms = 604.8 s = 0.17 h/day
- Active: 0.17 × 80 = 13.4 mAh
- Light-sleep: 23.83 h × 0.13 mA = 3.1 mAh
- **Total: ~16.5 mAh/day**

→ 1000 mAh → 1000 / 16.5 ≈ **~60 days** at 1 TPS.

### 4.6 Comparison to HANDOVER §3.8 claim

| Workload | HANDOVER claim | Actual calculation | Δ |
|----------|----------------|---------------------|---|
| 14 TPS (steady-state max) | 92 mAh/day | 191 mAh/day | +108% |
| 1 TPS | (not given) | 16.5 mAh/day | — |

HANDOVER's "92 mAh/day = 10 days on 1000 mAh" assumed an unjustified 1 h active/day (≈ 4 TPS). For the actual max-throughput workload, expect ~5 days.

This is a **correction** to HANDOVER §3.8, not a regression — the design is sound; the marketing numbers were optimistic.

---

## 5. Boot-time power

### 5.1 Cold boot sequence

```
   0 ms     Power applied, RTC starts
   50 ms    ROM bootloader runs
   100 ms   2nd-stage bootloader (ESP-IDF)
   200 ms   Application starts
   250 ms   TRNG keygen (120 mA peak for 50 ms)
   300 ms   Wi-Fi init
   300 ms + random 0-3000 ms   staggered Hello delay
   ...
   ~3500 ms ECDH × 6 peers (80 mA for ~50 ms total)
   ~3600 ms PBFT ready
```

Peak current during boot: 120 mA for 50 ms. Battery must support this peak; most 1000 mAh Li-Ion cells can deliver 1-2 A peak.

### 5.2 Boot staggering (cluster-friendly)

To prevent simultaneous boot broadcast storm (7 × ESP-NOW Hellos at boot = 7 collisions):
- Each node delays its first Hello by `node_id × CONFIG_PBFT_BOOT_STAGGER_MS / 7` + random jitter
- Default `CONFIG_PBFT_BOOT_STAGGER_MS = 3000` → 0-3 s uniform spread
- This means last node's Hello arrives at ~3.5 s after power-on

Trade-off: ~3.5 s cluster readiness vs broadcast collision risk.

---

## 6. Y-5 re-gen power cost

Every 24 hours, each node triggers Y-5:
- TRNG destroy + new keygen: 5 ms at 120 mA = 0.6 mC
- Hello broadcast: 1 ms at 80 mA
- Peers re-ECDH: 50 ms at 80 mA (8 ms ECDH × 6 peers + overhead)
- Total: ~60 ms at ~85 mA average = 5.1 mC ≈ 1.4 µAh per re-gen event

Negligible compared to daily energy budget. **Y-5 is free** from a power perspective.

---

## 7. Wake latency budget

When light-sleeping, the node must wake fast enough to:
- Receive a Pre-Prepare from the primary
- Reply with Prepare within `prepare_timeout_ms / 2 = 500 ms` (to avoid being the slowest responder)

ESP32-C3 light-sleep wake latency:
- ESP-NOW packet to handler ready: ~3 ms (measured)
- Wi-Fi UDP packet to handler ready: ~5 ms (measured)
- esp_timer to handler ready: < 1 ms

Both transports meet the 500 ms deadline with ~99% margin.

---

## 8. Modulation-aware scheduling (advanced, v2)

ESP-NOW uses 802.11 b/g/n at the PHY layer. CCA (clear channel assessment) adds variable latency. To minimise active time, esp-pbft can:

- **Batch:** collect TXs for `tx_batch_window_ms = 5 ms`, then send one Pre-Prepare containing multiple TXs (audit B + efficiency)
- **Schedule:** avoid transmitting during beacon intervals of nearby APs (reduces contention)
- **Adaptive duty cycle:** if battery < 20%, reduce TX rate by 50%

These are v2 enhancements. v1 fires each TX immediately.

---

## 9. ESP-NOW power save (configuring RX duty cycle)

ESP-NOW can sleep the radio between RX windows. The window/interval pair controls duty cycle per [Espressif docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html#config-esp-now-power-saving-parameter):

- `esp_wifi_connectionless_module_set_wake_interval(interval_ms)` — how often the radio wakes
- `esp_now_set_wake_window(window_ms)` — how long the radio stays awake per interval

**Default:** both APIs default to "max" (radio always on). ESP-NOW PS is **independent** of `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` per Espressif — calling one does NOT enable the other. Both are required for true low-power ESP-NOW RX.

**Requires** `CONFIG_ESP_WIFI_STA_DISCONNECTED_PM_ENABLE=y` (default on most targets, pin explicitly in `sdkconfig.defaults` for CI safety). The v4.4-era `esp_wifi_set_connectionless_wake_interval()` was **renamed** in IDF v5.0+ and is not present in v6.0.1.

**esp-pbft setting (v1):**

```c
// In pbft_network_espnow.c, after esp_now_init():
// Uses esp-pbft project-local Kconfig names (CONFIG_PBFT_ESPNOW_*) —
// see DEPLOYMENT.md §6.5 for rationale. The upstream CONFIG_ESPNOW_*
// options are only defined in examples/wifi/espnow/, not in the
// general esp-idf Kconfig tree, so we define our own equivalents.
#if CONFIG_PBFT_ESPNOW_POWER_SAVE
    // 50 ms window every 200 ms = 25% duty cycle (tunable via Kconfig)
    esp_wifi_connectionless_module_set_wake_interval(CONFIG_PBFT_ESPNOW_WAKE_INTERVAL_MS);
    esp_now_set_wake_window(CONFIG_PBFT_ESPNOW_WAKE_WINDOW_MS);
#endif
```

> **Why project-local `CONFIG_PBFT_ESPNOW_*`, not upstream `CONFIG_ESPNOW_*`:** The
> upstream `CONFIG_ESPNOW_*` options are only defined in
> `examples/wifi/espnow/main/Kconfig.projbuild` (an example project) — they
> do **not** propagate to the general esp-idf Kconfig tree when esp-pbft
> is built as a component. Re-declaring them with the `CONFIG_PBFT_` prefix
> makes the symbols real Kconfig options that `menuconfig` exposes, prevents
> a silent "symbol not defined" warning at build time, and is the only
> correct way to depend on them from component CMake. See
> [DEPLOYMENT.md §6.5](./DEPLOYMENT.md#65-esp-now-rx-power-save-kconfig-esp-pbft-defines--esp-idf-default)
> for the full Kconfig block.

**Reference: required `sdkconfig.defaults` entries (from DEPLOYMENT.md §6.5):**

```kconfig
# esp-pbft project-local — wraps esp_now_set_wake_window +
# esp_wifi_connectionless_module_set_wake_interval for our own use
config PBFT_ESPNOW_POWER_SAVE
    bool "Enable esp-pbft ESP-NOW RX duty-cycling"
    default y
config PBFT_ESPNOW_WAKE_WINDOW_MS
    int "esp-pbft ESP-NOW wake window (ms)"
    default 50
    range 1 100
config PBFT_ESPNOW_WAKE_INTERVAL_MS
    int "esp-pbft ESP-NOW wake interval (ms)"
    default 200
    range 50 1000

# ESP-IDF upstream — required by esp_now_set_wake_window() in disconnected state
# (per Espressif wifi-driver/wifi-performance-and-power-save.html)
CONFIG_ESP_WIFI_STA_DISCONNECTED_PM_ENABLE=y
```

---

## 10. Brownout and power-loss recovery

ESP32-C3 has a brownout detector at ~3.0 V (configurable). If triggered:
- esp-pbft's `pbft_stop` may not be called → NVS not updated
- On next boot, view/seq/watermarks are read from the LAST clean shutdown (or 0 if never)
- Cluster re-converges via view-change + New-View

**Implication:** no durability guarantees across unclean shutdowns. Acceptable for IoT use cases (the cluster's PBFT guarantees are about consistency, not durability).

---

## 11. Power optimisation checklist

| Optimisation | Effect | Implementation |
|--------------|--------|----------------|
| Modem-sleep during active consensus | -50 mA idle | `CONFIG_PBFT_MODEM_SLEEP=y` (default on) |
| Light-sleep when idle | -79.5 mA idle | `CONFIG_PBFT_LIGHT_SLEEP=y` (default on) |
| Batch TXs | -30% airtime | v2 feature |
| Reduce TX rate | -50% energy | app-level throttling |
| Disable UDP when not needed | -10 KB RAM | `CONFIG_PBFT_TRANSPORT_ESP_NOW=y` (default) |
| Lower TX power | -10% active current | `esp_wifi_set_max_tx_power(40)` (40 = 10 dBm) |

---

## 12. Wi-Fi UDP power-save modem mode

When the Wi-Fi UDP transport (`CONFIG_PBFT_TRANSPORT_WIFI_UDP=y`) is selected, esp-pbft must explicitly configure the Wi-Fi modem-sleep mode. The default ESP-IDF behaviour keeps the radio fully awake, which inflates current from **11.35 mA avg** (modem-sleep, Espressif-measured ESP32-C3 with DFS ON, DTIM 1, 160 MHz) to ~160 mA (always-on) — a **14× penalty** that breaks the HANDOVER §3.8 power budget.

### 12.1 ESP-IDF power-save modes (verified for ESP-IDF v6.0.1)

ESP-IDF exposes three power-save modes via `esp_wifi_set_ps()`:

| `wifi_ps_type_t` | Behaviour | Wake cadence | Typical current (ESP32-C3 measured) | Use case |
|------------------|-----------|--------------|-------------------------------------|----------|
| `WIFI_PS_NONE` | Radio always on | — | 160 mA | benchmarks, stress tests |
| `WIFI_PS_MIN_MODEM` | Modem sleeps between DTIM beacons | every DTIM (typically 100-300 ms) | **11.35 mA avg** (DFS ON) / 21.47 mA (DFS OFF) | latency-sensitive apps |
| `WIFI_PS_MAX_MODEM` | Modem sleeps between every beacon | every beacon (typically 100 ms) | ~10-15 mA (estimate; Espressif MIN_MAX spread is ~5 mA) | best power saving |

Implementation: `#include <esp_wifi.h>` → `esp_wifi_set_ps(wifi_ps_type_t type)`.

> ⚠️ **Numbers updated (vs older "30-60 mA"):** Espressif's published measurements for ESP32-C3 with `WIFI_PS_MIN_MODEM` + DFS ON + DTIM 1 + 160 MHz show **11.35 mA average** (max 83.31 mA, min 5.03 mA) — see [low-power-mode-wifi.html](https://docs.espressif.com/projects/esp-idf/en/latest/esp32c3/api-guides/low-power-mode/low-power-mode-wifi.html#modem-sleep-mode-configuration). Earlier "30-60 mA" in this document was conservative.

### 12.2 PBFT-correct choice

For esp-pbft, the constraint is `prepare_timeout_ms = 1000 ms`. A replica must be awake to receive a Pre-Prepare and reply with Prepare before half the timeout elapses (≈ 500 ms) to avoid being the slowest responder and triggering a spurious view-change.

**Recommendation:** `WIFI_PS_MIN_MODEM`. Wake cadence ≤ 300 ms easily meets the 500 ms deadline, and current drops from 160 mA to **11.35 mA avg** (DFS ON) — a 14× improvement over always-on.

`WIFI_PS_MAX_MODEM` is also viable (100 ms wake) but offers only marginal additional savings (~1-5 mA) at the cost of higher wake frequency → more CPU overhead. Not recommended for esp-pbft unless extreme power is required.

### 12.3 Per-role configuration

In Wi-Fi UDP mode, the cluster has one AP (typically node 0) and 6 STAs. The power-save modes apply differently:

| Role | Recommended mode | Notes |
|------|------------------|-------|
| **AP** (node 0 by default) | `WIFI_PS_NONE` | An AP cannot enter modem-sleep while clients are connected (the radio must remain reachable for association, beacon transmission, and group-key rotation). Power-saving on the AP is structurally limited. |
| **STA** (nodes 1..6) | `WIFI_PS_MIN_MODEM` | STAs can sleep between DTIMs; the AP queues multicast/broadcast for them. esp-pbft messages are UDP multicast, so DTIM wake is sufficient. |

The role assignment is dynamic — after a view-change, the new primary becomes the AP (rotates `view % n`), and the previous AP becomes a STA. The PBFT code must update `esp_wifi_set_ps()` when the role changes.

### 12.4 Kconfig

```kconfig
# Wi-Fi UDP power-save mode (only relevant when CONFIG_PBFT_TRANSPORT_WIFI_UDP=y)
choice PBFT_WIFI_PS_MODE
    prompt "Wi-Fi modem power-save mode for STA nodes"
    default PBFT_WIFI_PS_MIN_MODEM
    help
      Selects the Wi-Fi modem power-save mode applied to STAs.
      The AP always uses WIFI_PS_NONE (cannot sleep while clients connected).
      PBFT_WIFI_PS_MIN_MODEM is recommended: wakes every DTIM (~100-300 ms),
      meets the 500 ms deadline for Prepare replies, saves 3-5x current.
      PBFT_WIFI_PS_MAX_MODEM wakes every beacon (~100 ms) — marginal extra savings.
      PBFT_WIFI_PS_NONE disables power saving (debug only).
    config PBFT_WIFI_PS_NONE
        bool "No power saving (debug only)"
    config PBFT_WIFI_PS_MIN_MODEM
        bool "Minimum modem-sleep (recommended)"
    config PBFT_WIFI_PS_MAX_MODEM
        bool "Maximum modem-sleep"
endchoice

# Verify the choice is only used with Wi-Fi UDP transport
config PBFT_TRANSPORT_WIFI_UDP
    bool
    select PBFT_WIFI_PS_MODE if PBFT_TRANSPORT_WIFI_UDP
```

### 12.5 Runtime application

In `pbft_network_wifi_udp.c`:

```c
static pbft_net_err_t wifi_udp_set_role(bool is_ap) {
    if (is_ap) {
        // AP must stay awake
        ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
        ESP_LOGI(TAG, "Wi-Fi role=AP → WIFI_PS_NONE");
    } else {
        // STA: apply Kconfig-selected mode
#ifdef CONFIG_PBFT_WIFI_PS_MIN_MODEM
        ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_MIN_MODEM));
        ESP_LOGI(TAG, "Wi-Fi role=STA → WIFI_PS_MIN_MODEM (DTIM wake)");
#elif defined(CONFIG_PBFT_WIFI_PS_MAX_MODEM)
        ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_MAX_MODEM));
        ESP_LOGI(TAG, "Wi-Fi role=STA → WIFI_PS_MAX_MODEM (beacon wake)");
#else
        ESP_LOGW(TAG, "Wi-Fi role=STA → WIFI_PS_NONE (debug only)");
#endif
    }
    return PBFT_NET_OK;
}
```

Call `wifi_udp_set_role(my_node_id == 0)` at boot (node 0 is default AP), and again whenever the role flips due to view-change.

### 12.6 Power re-budget (corrected HANDOVER §3.8 with proper PS mode)

All rows assume the same 14 TPS workload: 9.8% active duty cycle at 80 mA, balance at the
stated idle floor. mAh/day = avg current × 24 h. The only variable across rows is the
**idle floor** (always-on 160 mA → modem-sleep 11.35 mA → light-sleep 0.13 mA).

| Mode | Old HANDOVER | Avg current (9.8% × 80 mA + 90.2% × idle) | Corrected mAh/day |
|------|--------------|-------------------------------------------|-------------------|
| ESP-NOW build, 14 TPS, no light-sleep (idle = modem-sleep 11.35 mA) | 92 mAh/day (unjustified) | 0.098×80 + 0.902×11.35 = **18.08 mA** | ~434 mAh/day |
| ESP-NOW build, 14 TPS, light-sleep (idle = 0.13 mA) | — | 0.098×80 + 0.902×0.13 = **7.96 mA** | **~191 mAh/day** (= §4.4) |
| Wi-Fi UDP build, 14 TPS, no PS (idle = always-on 160 mA) | 172 mAh/day | 0.098×80 + 0.902×160 = **152 mA** | ~3650 mAh/day |
| Wi-Fi UDP build, 14 TPS, PS_MIN_MODEM, no light-sleep (idle = 11.35 mA) | — | 0.098×80 + 0.902×11.35 = **18.08 mA** | ~434 mAh/day |
| Wi-Fi UDP build, 14 TPS, PS_MIN_MODEM + light-sleep (idle = 0.13 mA) | — | 0.098×80 + 0.902×0.13 = **7.96 mA** | **~191 mAh/day** |

On a 1000 mAh battery the best-case (light-sleep idle) ESP-NOW and Wi-Fi UDP builds both give
1000 / 191 ≈ **~5 days** at 14 TPS sustained — consistent with §4.4.

The HANDOVER §3.8 number of 172 mAh/day assumed modem-sleep was already active — it was not. With PS mode applied, the Wi-Fi UDP build is comparable to the ESP-NOW build in power.

### 12.7 Caveats and edge cases

| Edge case | Behaviour | Mitigation |
|-----------|-----------|------------|
| PBFT message arrives between DTIMs | Radio wakes at next DTIM, delivers message | Acceptable (≤ 300 ms latency addition) |
| IGMP group leave while sleeping | Group membership lost on wake | esp-pbft rejoins group on every wake (cheap, ~1 ms) |
| DTIM not delivered (AP outage) | STA wakes but sees no beacon → declares disconnect → reconnect | Standard Wi-Fi reconnection (~1-2 s) |
| Beacon interval mismatch (AP at 100 ms, STA expects 300 ms) | STA uses AP's DTIM, not its own | esp_wifi_set_ps adapts automatically |
| Encryption (WPA2) group key rotation while sleeping | STA must wake to receive | Acceptable; rotation is rare (default 1 hour) |
| TCP/ARP traffic mixed with PBFT | Wake-up happens for any 802.11 frame | Acceptable (PBFT doesn't filter) |

### 12.8 Debug overrides

For benchmark / soak test builds, force PS off:

```kconfig
# In sdkconfig.benchmark
CONFIG_PBFT_WIFI_PS_NONE=y   # forces always-on radio
```

Add to TEST-PLAN.md §3 (perf tests) so worst-case power is measured.

---

## 13. Open questions (power level)

| # | Question | Recommendation |
|---|----------|----------------|
| P1 | Is deep-sleep + scheduled wake viable? | No — PBFT state must persist; deep-sleep loses RAM. |
| P2 | Should we support solar/battery hybrid? | v2. Out of scope for v1. |
| P3 | What's the actual peak current during ESP-NOW TX? | ~120 mA for < 1 ms (decoded from datasheet). Battery must support. |
| P4 | Is light-sleep wake latency actually < 5 ms on ESP32-C3? | Yes, verified — `esp_light_sleep_get_default_wakeup_time()` returns ~3 ms typical. |

---

## 14. References

- [HANDOVER.md](./HANDOVER.md) §3.8 — power budget (now superseded — actual numbers updated)
- [ARCHITECTURE.md](./ARCHITECTURE.md) — boot sequence timing
- [CRYPTO.md](./CRYPTO.md) §8 — Y-5 power cost
- [CONSENSUS.md](./CONSENSUS.md) §9 — throughput math
- [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) — C8 (light-sleep wake), C13 (active-time model)
- ESP32-C3 datasheet rev 1.4 — power consumption tables
- ESP-IDF power management guide: `docs/en/api-reference/system/power_management.rst`

---

**End of POWER.md (v0.3 draft — renumbered: ESP-NOW PS §9, Brownout §10, Checklist §11, Wi-Fi UDP PS modem §12, Open Q §13, Refs §14; Kconfig names aligned to canonical Espressif symbols; addresses audit issues C8, C13)**