# Diagnostic Guide: `Failed to connect to the adapter (Error: SRSP - SYS - ping after 6000ms)` & `SYS_RESET_IND`

**Affected Platforms (Stock Firmware):** TI Z-Stack 3.x.0 (`CC2652P`, `CC2652RB`, `CC1352P7` / `CC2652P7`, `CC2674P10`) on Zigbee2MQTT (`zigbee-herdsman`) and Home Assistant ZHA (`zigpy-znp`).  
**Resolved In:** **[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** (`SLZB-06`, `SLZB-06U`, `SLZB-06P7` / `SLZB-06P7v1`, `SLZB-06P7U`, `SLZB-07P7`, `SLZB-06P10`, `SLZB-07P10`).  
**Official ZeroStack Page:** [`https://smlight.tech/software/zerostack`](https://smlight.tech/software/zerostack)  
**Interactive Crash & Airtime Analyzer:** [`https://smlight-tech.github.io/zerostack/`](https://smlight-tech.github.io/zerostack/)  
**AI Automated Triage:** [`github.com/zerostack-rf/zerostack-mcp`](https://github.com/zerostack-rf/zerostack-mcp)

---

## 1. Exact Verbatim Log Signatures

This failure occurs when restarting the Zigbee2MQTT container, reloading the ZHA integration, reconnecting over TCP/UART after a soft reset (`SYS_RESET_REQ`), or when the coordinator spontaneously reboots mid-operation (`SYS_RESET_IND`):

### Zigbee2MQTT / `zigbee-herdsman`
```text
Zigbee2MQTT:info  2026-10-24 12:00:01: Starting Zigbee2MQTT version 2.1.1 (commit #abcdef)
Zigbee2MQTT:info  2026-10-24 12:00:01: Starting zigbee-herdsman (3.2.0)
zh:zstack:znp: Opening TCP socket with 192.0.2.50:6638
zh:zstack:znp: Socket connected
zh:zstack:znp: Writing CC2530/CC2531 skip bootloader payload
zh:zstack:znp:SREQ: --> SYS - ping - {}
Zigbee2MQTT:error 2026-10-24 12:00:19: Error while starting zigbee-herdsman
Zigbee2MQTT:error 2026-10-24 12:00:19: Failed to start zigbee
Zigbee2MQTT:error 2026-10-24 12:00:19: Exiting...
Zigbee2MQTT:error 2026-10-24 12:00:19: Error: Failed to connect to the adapter (Error: SRSP - SYS - ping after 6000ms)
    at ZStackAdapter.start (/app/node_modules/zigbee-herdsman/src/adapter/z-stack/adapter/zStackAdapter.ts:113:27)
    at Controller.start (/app/node_modules/zigbee-herdsman/src/controller/controller.ts:136:29)
```

Spontaneous mid-operation coordinator crash / watchdog reset:
```text
zh:zstack:znp:AREQ: <-- SYS - resetInd - {"reason":2,"transportrev":2,"productid":1,"majorrel":2,"minorrel":7,"hwrev":0}
Zigbee2MQTT:error: Adapter disconnected, restarting... (Received unexpected reset from adapter: SYS_RESET_IND)
```

### Home Assistant ZHA (`zigpy-znp`)
```text
Logger: homeassistant.components.zha.core.gateway
Source: components/zha/core/gateway.py:175
Config entry 'SLZB-06' for zha integration not ready yet: Failed to connect to ZNP radio: SRSP not received for SYS.Ping.Req within 6.00s; Retrying in background
```

---

## 2. Silicon, UART & NV Root Cause Analysis (Why Stock Firmware Fails Warm Reconnects & Reboots)

Hardware validation measurements on `TI CC1352P7` @ `48.0 MHz` (`DWT->CYCCNT`) identify three root causes behind `SRSP - SYS - ping after 6000ms` connect failures and unexpected `SYS_RESET_IND` reboots on stock TI Z-Stack 3.x.0:

1. **8.84-Second NPI UART Serial Blackout (`44` Lost Pings) on Delimiter/Bootloader Noise**:
   - When `zigbee-herdsman` or `zigpy-znp` connects, it transmits a legacy skip-bootloader magic byte (`0xEF`) or reconnects mid-frame, followed immediately by `SYS_PING` (`FE 00 21 01 20`).
   - Hardware UART fault-injection testing (`115,200` baud across 6 hostile framing scenarios) proves that when legacy Z-Stack's NPI parser receives a `300 × SOF (0xFE)` delimiter burst or unconstrained line noise, its parser state machine wedges for **`8,840 ms` (`8.84 seconds`, losing `44` consecutive pings)**—exceeding Zigbee2MQTT's 6,000 ms startup timeout. Even a single truncated frame costs `203 ms` (`1` lost ping).
2. **Blocking `NVOCMP` Flash Compaction (`163.9 ms` / Page) & Torn-Record Brownout Storms (`20,232 B` Written)**:
   - On legacy Z-Stack, every unchanged 100-byte state write appends to flash (`100,714` cycles), triggering a **`163.9 ms` flash compaction stall** every 69 writes with interrupts masked.
   - Worse, if a power dip or brownout interrupts a flash write (leaving a torn record), legacy Z-Stack falls back to a **full 62-page flash compaction on reboot—writing `20,232 bytes` to flash and executing `62` sector erases** before `NPITask` can service the serial port.
3. **Deterministic Remote Over-the-Air CPU `HardFault` (`SYS_RESET_IND`)**:
   - On-target fault injection across 12 malformed secured NWK/APS frame classes demonstrates that an unauthenticated over-the-air NWK frame with a truncated AuxHeader, Payload, or MIC causes a **deterministic CPU `HardFault` (`Precise BusFault`) on stock Z-Stack after `143,345–144,412` cycles (`~3.0 ms`)**, immediately rebooting the coordinator and emitting `AREQ - SYS - resetInd`.

---

## 3. Immediate Mitigations for Legacy / Stock Firmware Users

1. **Match `rtscts` to Your SMLIGHT Hardware Model**:
   - **SLZB-06U**, **SLZB-06P7U**, and **SLZB-06P10** have hardware RTS/CTS wired (`DIO10` RTS / `DIO11` CTS).
   - **SLZB-06**, **SLZB-06P7**, **SLZB-06P7v1**, **SLZB-07P7**, and **SLZB-07P10** do not wire RTS/CTS—always set `rtscts: false` in `configuration.yaml` on these models and on TCP socket bridges.
2. **Wait 10 Seconds Between Container Restarts on Stock Firmware**: Allow any in-flight NPI frame parser timeout or `NVOCMP` compaction to finish before reconnecting.

---

## 4. How SMLIGHT ZeroStack Permanently Eliminates This Fault

**[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** eliminates warm-reset timeouts, serial blackouts, brownout corruption, and remote OTA crashes across **SLZB-06**, **SLZB-06U**, **SLZB-06P7** / **SLZB-06P7v1**, **SLZB-06P7U**, **SLZB-07P7**, **SLZB-06P10**, and **SLZB-07P10**:

| Resilience / Serial / Boot Metric | Stock TI Z-Stack (`DWT->CYCCNT`) | SMLIGHT ZeroStack (`Release Zero 0.9.9-beta.1`) | Hardware-Measured Improvement |
| :--- | :--- | :--- | :--- |
| **Hostile UART Serial Framing (6 scenarios: `300 × SOF`, noise, truncated)** | **`8.84 s` blackout (`44` lost pings)** on `300 × SOF`/noise; `1` lost ping (`203 ms`) on truncated | **`0` lost pings across all 6 scenarios (`3–55 ms` wire-speed recovery; `29 ms` on `300 × SOF`, `3 ms` on truncated)** | **Zero serial blackouts (`0` lost pings vs `44`)** |
| **Host Serial `SYS_PING` Round-Trip (`115,200` baud)** | `9,790` cyc (`4,921` cyc reply path) | **`4,618` cyc (`816` cyc reply path) + Live Auto-Baud (`115200`/`460800`/`1000000`)** | **83% faster reply path (53% fewer total cycles)** |
| **NOR Flash Brownout Fuzzing (`22,959` hardware power cuts + `2M` shadow ops)** | `1` NOR verify conflict; torn-record reboot writes **`20,232 B` (`62` page erases)** | **`0` lost/corrupted records, `0` NOR conflicts**; torn-record reboot writes **`27 B` (`0` page erases)** | **749× lower post-brownout reboot flash I/O** |
| **Idempotent NVM State Write & Compaction Rate** | `100,714` cyc; compacts at **write #69 (`163.9 ms` stall)** | **`3,770` cyc (`0 B` flash writes); compacts at write #1,013 (`13.7 ms` stall)** | **26.7× faster write + 14.7× lower compaction rate** |
| **Malformed Secured OTA NWK/APS Frames (12 fault classes)** | **Deterministic remote CPU `HardFault` (`Precise BusFault`)** after `143,345–144,412` cyc | **`0` crashes across all 12 classes & 3,255 MAC vectors** (rejected in `320–6,631` cyc) | **Remote OTA `HardFault` completely eliminated** |
| **1,000-Frame Cryptographic Replay Flood (D=150)** | `143,802` cyc (`3.0 ms`)/replay (**`2.99 s` total CPU freeze**) | **`1,550` cyc (`32 µs`)/replay (`32.4 ms` total CPU)** | **92.8× faster replay flood rejection** |

### Detailed Hardware Results: Adversarial UART Serial Framing Abuse (`115,200` Baud)

| Adversarial UART Input Scenario | Legacy Reference Stack: Pings Lost | Legacy Reference Stack: Recovery Time | SMLIGHT ZeroStack: Pings Lost | SMLIGHT ZeroStack: Recovery Time | SWD Heap Δ (Both Arms) |
| :--- | ---: | ---: | ---: | ---: | ---: |
| **512 B random noise (excluding `SOF`)** | `0` | `48 ms` | **`0`** | **`47 ms`** | `0 B` |
| **10 × `PING` frames with corrupted FCS checksum** | `0` | `6 ms` | **`0`** | **`9 ms`** | `0 B` |
| **Max-length header + 256 B garbage body** | `0` | `25 ms` | **`0`** | **`25 ms`** | `0 B` |
| **Truncated frame (cut mid-payload)** | **`1` (every rep)** | **`203 ms`** | **`0`** | **`3 ms`** | `0 B` |
| **SOF Delimiter Storm (`300 × SOF` back-to-back)** | **`44` (every rep)** | **`8,840 ms` (`8.84 s`)** | **`0`** | **`29 ms`** | `0 B` |
| **512 B unconstrained random noise (including `SOF`)** | **`30`, `4`, then `0`** | **`6,000 ms` (`6.0 s`)** | **`0`** | **`47–55 ms`** | `0 B` |

### Detailed Hardware Results: NOR Flash Power-Loss & Brownout Recovery (`22,959` Cut Points)

| Brownout Scan / Post-Reboot Scenario | Legacy Reference Stack | SMLIGHT ZeroStack (`Release Zero 0.9.9-beta.1`) | Engineering Outcome |
| :--- | :--- | :--- | :--- |
| **Mid-Compaction Power Cut (64 variable records, 32 updated, 1 deleted)** | `20,511` cuts (`0` lost, **`1` NOR hardware write-verify conflict**) | **`22,565` cuts (`0` lost, `0` NOR conflicts, `0` boot errors)** | **100% power-fail-safe compaction across all `22,565` cuts.** |
| **Torn Tail, Large Record (`380` byte-level cuts) & Small Record (`14` cuts)** | `394` cuts (`0` torn payloads) | **`394` cuts (`0` torn payloads)** | Every cut resolves deterministically to intact prior or new record. |
| **Boot Recovery After Mid-Compaction Cut (Flash Bytes Written / Erases)** | `19,377 B` mean (`22,458 B` max) / `62–64` erases | **`19,405 B` mean (`20,529 B` max) / `56–63` erases** | Clean single-pass ring recovery without redundant page loops. |
| **Boot Recovery After Torn Record Tail (Flash Bytes Written / Erases)** | **`20,232 B` mean (`20,447 B` max) / `62–63` page erases** | **`27 B` mean (`27 B` max) / `0` page erases** | **749× lower post-brownout boot flash write volume (`0` page erases vs `62`).** |
