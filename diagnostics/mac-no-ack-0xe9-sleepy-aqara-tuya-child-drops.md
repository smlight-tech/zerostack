# Diagnostic Guide: `MAC_NO_ACK (0xe9)` / `status=233`, `0xf0: MAC_TRANSACTION_EXPIRED`, `0xf1: MAC_TRANSACTION_OVERFLOW` & Sleepy Aqara / Tuya Child Drops

**Affected Platforms (Stock Firmware):** TI Z-Stack 3.x.0 (`CC2652P`, `CC2652RB`, `CC1352P7` / `CC2652P7`, `CC2674P10`) and Silicon Labs EmberZNet (`EFR32MG21` / `EFR32MG24`) on Zigbee2MQTT and Home Assistant ZHA.  
**Resolved In:** **[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** for SMLIGHT TI SimpleLink F2 coordinators (`SLZB-06`, `SLZB-06U`, `SLZB-06P7` / `SLZB-06P7v1`, `SLZB-06P7U`, `SLZB-07P7`, `SLZB-06P10`, `SLZB-07P10`; *Silicon Labs EFR32 support is in active development*).  
**Official ZeroStack Page:** [`https://smlight.tech/software/zerostack`](https://smlight.tech/software/zerostack)  
**Interactive Crash & Airtime Analyzer:** [`https://smlight-tech.github.io/zerostack/`](https://smlight-tech.github.io/zerostack/)  
**AI Automated Triage:** [`github.com/zerostack-rf/zerostack-mcp`](https://github.com/zerostack-rf/zerostack-mcp)

---

## 1. Exact Verbatim Log Signatures

Users notice battery-powered Zigbee End Devices (ZEDs)—especially Aqara / Xiaomi LUMI sensors, Sonoff SNZB sensors, Tuya battery buttons/TRVs, and IKEA remotes—silently dropping offline after 1–2 hours, failing interview, or throwing `0xe9` (`233`), `0xf0` (`240`), or `0xf1` (`241`) when queried or configured:

### A. `0xe9: MAC_NO_ACK` (`status=233`)
```text
ERROR: Publish 'set' 'state' to 'Aqara_Contact_Sensor' failed: 'Error: Data request failed with error: 'MAC no ack' (233)'
zh:zstack:znp:AREQ: <-- AF - dataConfirm - {"status":233,"endpoint":1,"transid":42}
Error: Data request failed with error: 'MAC no ack' (233)
    at ZStackAdapter.sendZclFrameToEndpointInternal (/app/node_modules/zigbee-herdsman/src/adapter/z-stack/adapter/zStackAdapter.ts:439:23)
```

```text
zigpy.exceptions.DeliveryError: Failed to deliver message: <ZdoStatus.MAC_NO_ACK: 233>
zigpy.exceptions.DeliveryError: [0x8a42:1:0x0006]: Message send failure: <sl_Status.MAC_NO_ACK_RECEIVED: 0xe9>
```

### B. `0xf0: MAC_TRANSACTION_EXPIRED` (`status=240`)
```text
ERROR: Publish 'set' 'occupied_heating_setpoint' to 'Bedroom_TRV' failed: 'Error: Data request failed with error: 'MAC transaction expired' (240)'
zh:zstack:znp:AREQ: <-- AF - dataConfirm - {"status":240,"endpoint":1,"transid":89}
zigpy.exceptions.DeliveryError: Failed to deliver message: <Status.MAC_TRANSACTION_EXPIRED: 240> (0xf0)
```

### C. `0xf1: MAC_TRANSACTION_OVERFLOW` (`status=241`)
```text
ERROR: Publish 'get' 'state' to 'Window_Sensor_14' failed: 'Error: Data request failed with error: 'MAC transaction overflow' (241)'
zh:zstack:znp:AREQ: <-- AF - dataConfirm - {"status":241,"endpoint":1,"transid":90}
zigpy.exceptions.DeliveryError: Failed to deliver message: <Status.MAC_TRANSACTION_OVERFLOW: 241> (0xf1)
```

---

## 2. Silicon & MAC Root Cause Analysis (`0xe9`, `0xf0`, and `0xf1`)

Sleepy Zigbee End Devices (ZEDs) keep their receivers turned off (`RxOnWhenIdle = FALSE`) to conserve coin-cell battery life. When the coordinator wants to send a packet to a direct sleepy child, it cannot transmit immediately; it must place the frame into the **MAC Indirect Transmission Queue** (`macData.txQueue`) and wait for the sleepy ZED to wake up and transmit an IEEE 802.15.4 `MAC_DATA_REQ` (Data Poll) command frame (`FCF = 0x8863`).

Under IEEE 802.15.4-2020, the coordinator must reply with a hardware Auto-ACK within **`aTurnaroundTime` (192 µs = 12 symbols)** of the final byte of the `MAC_DATA_REQ`. Inside that 192 µs window, the coordinator's MAC hardware must set the **Frame Pending (`FP`, bit 4 of FCF byte 0)** flag accurately:
- **`FP = 1`**: Tells the sleepy ZED *"Stay awake for up to `macMaxFrameTotalWaitTime`—I have an indirect packet queued for you."*
- **`FP = 0`**: Tells the sleepy ZED *"Nothing queued—turn off your radio receiver immediately within <1 ms."*

### 1. Why `0xe9: MAC_NO_ACK` (`status=233`) Happens on Sleepy ZEDs
Because software on the main MCU cannot parse the `MAC_DATA_REQ`, walk the NWK indirect queue, and construct the ACK within 192 µs under arbitrary RTOS interrupt latency, the SimpleLink F2 Cortex-M0 RF Core (`CC2652P`, `CC2652P7` / `CC1352P7`, `CC2674P10`) maintains a **32-slot Hardware Source Matching (`SrcMatch`) Table** inside Radio RAM (`RFC_RAM:extRam.srcMatchShortEntry`):
1. **Hardware `FP = 1` Auto-ACK vs. Upper-MAC Dequeue Turnaround**: When a sleepy child wakes up and transmits `MAC_DATA_REQ` (`0x04`), the Cortex-M0 CPE checks `SrcMatch` (or `pendAll` when >32 direct sleepy ZEDs are present), sets **`FP = 1`** in the 192 µs hardware Auto-ACK, and notifies the upper software MAC on the main Cortex-M4F/M33 CPU to dequeue, encrypt, and transmit the held indirect frame.
2. **Flash-Stall Poll-Window Miss (`11.11 ms` NWK Security & `163.9 ms` `NVOCMP` Stalls)**: Seeing `FP = 1`, the sleepy ZED keeps its receiver awake only for a brief `macMaxFrameTotalWaitTime` RX poll window before powering down to conserve battery. On stock TI Z-Stack, O(N⋅P) per-frame flash security walks (`533,156` cycles = **`11.11 ms`** per NWK encrypt at D=150; `1,486,228` cycles = **`30.96 ms`** per TCLK miss), **`163.9 ms`** `NVOCMP` flash page compactions with interrupts masked, and **`3.00 s`** broadcast echo storms stall the upper MAC so the indirect frame is dequeued and transmitted *after* the sleepy ZED's RX poll window has already closed—transmitting into dead air and failing after 4 MAC retries with **`0xe9: MAC_NO_ACK` (`233`)**.
3. **Beyond-32 `pendAll` Contention & `nwkDB` Buffer Hoarding**: Beyond 32 direct sleepy ZEDs, `pendAll` forces `FP = 1` on every poll ACK, keeping idle ZEDs awake unnecessarily while offline or slow-polling ZEDs can hoard nearly 100% of the shared `nwkDB` buffer pool (`69 / 72` buffers), starving active mesh traffic.
4. **Aqara / LUMI Early Disassociation**: Legacy Xiaomi/Aqara sensors treat repeated missed `MAC_DATA_REQ` poll extractions or unacknowledged indirect frames during check-in/rejoin as parent loss and permanently stop polling without attempting a standard Zigbee 3.0 Trust Center rejoin.

### 2. Why `0xf0: MAC_TRANSACTION_EXPIRED` (`status=240`) Happens
`0xf0: MAC_TRANSACTION_EXPIRED` (`MAC_TRANSACTION_EXPIRED = 240`) occurs when an indirect frame sits in the coordinator's `macData.txQueue` for the entire indirect persistence window (`macTransactionPersistenceTime` / `NWK_INDIRECT_MSG_TIMEOUT`, typically **7.68 seconds** in stock Z-Stack) without the sleepy ZED completing a successful `MAC_DATA_REQ` poll extraction.
- **Fast-Polling vs. Slow-Polling Mismatch & Poll-Window Stalls**: Many Tuya TRVs, IKEA buttons, and Aqara sensors sleep for 30–300 seconds between normal polls (`Long Poll Interval`), only switching to `Fast Poll` (every 250 ms) during an active ZCL transaction or immediately after waking up. If the host sends an unsolicited command while the ZED is in its deep sleep window—or if an `11.11 ms` flash security walk or `163.9 ms` `NVOCMP` compaction delayed upper-MAC indirect dequeue past the ZED's `FP = 1` RX poll window—the indirect packet expires after 7.68 seconds and fails with **`0xf0: MAC_TRANSACTION_EXPIRED`**.

### 3. Why `0xf1: MAC_TRANSACTION_OVERFLOW` (`status=241`) Happens
`0xf1: MAC_TRANSACTION_OVERFLOW` (`MAC_TRANSACTION_OVERFLOW = 241`) is returned when the lower MAC indirect transmission queue (`MAC_CFG_TX_DATA_MAX` / `MAC_CFG_TX_MAX`, typically only **5 to 8 frames** in stock F2 builds) or the 32-slot `SrcMatch` pending table is completely full when the upper NWK layer attempts to enqueue another indirect packet for a sleepy child.
- Because stock firmware allows a single slow-polling ZED to consume all 5–8 indirect MAC queue slots during a ZDO interview or multi-attribute configuration burst (holding each slot for 7.68 seconds until `0xf0` expiry), any command to *any other* sleepy sensor during that window immediately fails with **`0xf1: MAC_TRANSACTION_OVERFLOW`**.

---

## 3. Immediate Mitigations for Legacy / Stock Firmware Users

1. **Force Sleepy Sensors Through AC Mains Routers**: If your coordinator runs stock firmware, keep direct sleepy children on `0x0000` below 20–25 devices by pairing battery sensors near AC-powered smart plugs.
2. **Trigger Configuration Only on Device Wake-Up**: For slow-polling battery devices (`0xf0`), queue configuration changes in Zigbee2MQTT so they dispatch immediately when the sensor sends an uplink attribute report or check-in.
3. **Avoid Bulk Queries to Battery Devices (`0xf1`)**: Never include battery-powered ZEDs in startup state-refresh scripts that enqueue multiple indirect transactions simultaneously.

---

## 4. How SMLIGHT ZeroStack Eliminates Sleepy-Device Bottlenecks (`0xe9`, `0xf0`, and `0xf1`)

While both stock TI Z-Stack and ZeroStack use the same SimpleLink F2 Cortex-M0 CPE hardware 32-slot `SrcMatch` table and hardware `FP=1` auto-ACK mechanism in `RFC_RAM`, **[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** eliminates upper-MAC indirect poll timeouts and buffer starvation across **SLZB-06**, **SLZB-06U**, **SLZB-06P7** / **SLZB-06P7v1**, **SLZB-06P7U**, **SLZB-07P7**, **SLZB-06P10**, and **SLZB-07P10**:

- **Zero Flash-Read Stalls During Sleepy Poll Windows (Eliminates Poll-Turnaround `0xe9` & `0xf0`)**:
  - After the Cortex-M0 RF Core replies to a sleepy child's `MAC_DATA_REQ` with `FP=1`, the main CPU must dequeue, encrypt, and transmit the held indirect packet before the sleepy sensor's receive window closes. On legacy Z-Stack, O(N⋅P) flash security scans (`11.11 ms` per NWK encrypt, `30.96 ms` per TCLK miss), `163.9 ms` `NVOCMP` compactions, and `3.00 s` broadcast echo stalls block the main CPU so the indirect frame is transmitted too late (after the ZED has already timed out and powered down its receiver, failing with `0xe9: MAC_NO_ACK`).
  - ZeroStack executes NWK encryption in **`82.7 µs` (`3,968` cycles, `0` flash reads — 134.4× faster)**, TCLK lookups in **`6.8 µs` (4,559× faster)**, and runs storage cleanup only when the radio is idle—ensuring queued indirect frames are dequeued and transmitted immediately while the polling ZED is still awake.
- **Indirect Queue Quota Isolation & O(1) 218-Cycle Memory Allocation (Eliminates `0xf1` & `0x11: BUFFER_FULL`)**:
  - Stock Z-Stack allows sleeping or offline ZEDs to hoard nearly 100% of the shared `nwkDB` buffer pool (`69 / 72` buffers), starving both other sleepy sensors (`0xf1: MAC_TRANSACTION_OVERFLOW`) and active mesh routers (`0x11: BUFFER_FULL`).
  - ZeroStack enforces strict per-child and global indirect queue quotas (`50%` pool cap, max `2` queued frames per sleeping child) backed by constant-time O(1) memory allocation (**`218` CPU cycles = `4.5 µs`, 11.7× faster**), ensuring unresponsive battery sensors can never exhaust coordinator memory or block active mesh traffic.
