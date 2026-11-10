# Diagnostic Guide: `Error: SRSP - AF - dataRequestExt after 6000ms`, TCLK Interview Timeouts & ZNP SREQ Lockups

**Affected Platforms (Stock Firmware):** TI Z-Stack 3.x.0 (`CC2652P`, `CC2652RB`, `CC1352P2`, `CC1352P7` / `CC2652P7`, `CC2674P10`) on Zigbee2MQTT (`zigbee-herdsman` `zstack` adapter) and Home Assistant ZHA (`zigpy-znp`).  
**Resolved In:** **[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** (`SLZB-06`, `SLZB-06U`, `SLZB-06P7` / `SLZB-06P7v1`, `SLZB-06P7U`, `SLZB-07P7`, `SLZB-06P10`, `SLZB-07P10` across `CC2652P`, `CC2652P7` / `CC1352P7`, and `CC2674P10`).  
**Official ZeroStack Page:** [`https://smlight.tech/software/zerostack`](https://smlight.tech/software/zerostack)  
**Interactive Crash & Airtime Analyzer:** [`https://smlight-tech.github.io/zerostack/`](https://smlight-tech.github.io/zerostack/)  
**AI Automated Triage:** [`github.com/zerostack-rf/zerostack-mcp`](https://github.com/zerostack-rf/zerostack-mcp)

---

## 1. Exact Verbatim Log Signatures

Users experiencing this fault see outbound commands stall for 6,000 ms (the default `zigbee-herdsman` SREQ timeout) or device pairing interviews fail during Trust Center Link Key (TCLK) exchange:

### Zigbee2MQTT / `zigbee-herdsman`
```text
zh:zstack:unpi:writer: --> frame [254,28,36,2,2,0,0,0,0,0,0,0,0,0,1,0,239,1,0,0,30,15,24,10,6,0,0,16,1,210]
zh:zstack:znp:SREQ: --> AF - dataRequest - {"dstaddr":19458,"destendpoint":1,"srcendpoint":1,"clusterid":6,"transid":21,"options":0,"radius":30,"len":3}
ERROR: Publish 'set' 'state' to 'Living_Room_Switch' failed: 'Error: Data request failed with error: 'SRSP - AF - dataRequest after 6000ms''
Error: SRSP - AF - dataRequestExt after 6000ms
    at Object.waitFor (/app/node_modules/zigbee-herdsman/src/utils/waitress.ts:63:23)
    at Znp.request (/app/node_modules/zigbee-herdsman/src/adapter/z-stack/znp/znp.ts:285:28)
```

Often accompanied by cascading ZDO, Active Endpoint, or TCLK interview timeouts:
```text
Error: SRSP - ZDO - activeEpReq after 6000ms
Error: SRSP - ZDO - nodeDescReq after 6000ms
Error: SRSP - UTIL - assocGetWithAddress after 6000ms
Error: Failed to interview '0xa4c138...': Request-Key timeout / 'APS no ack' (167) (0xa7)
```

### Home Assistant ZHA (`zigpy-znp`)
```text
zigpy_znp.exceptions.CommandNotRecognized: SRSP not received for AF.DataRequestExt.Req within 5.00s
zigpy.exceptions.DeliveryError: Failed to deliver message: <sl_Status.TIMEOUT: 6>
```

---

## 2. Silicon & RTOS Root Cause Analysis (Why Stock TI Z-Stack Fails)

In the TI Monitor & Test (MT / ZNP) serial protocol (`0xFE` SOF framing), every host-originated Synchronous Request (`SREQ`, `CMD0[7:5] == 0x01`) requires the coordinator to execute the command and return a Synchronous Response (`SRSP`, `CMD0[7:5] == 0x03`) before the host is allowed to transmit the next `SREQ`.

Hardware `DWT->CYCCNT` cycle-counter profiling on `TI CC1352P7` @ `48.0 MHz` (`1 µs = 48 cycles`) exposes four compounding bottlenecks in stock TI Z-Stack 3.x.0:

1. **O(N⋅P) Linear Flash Log Walks on Every Secured Frame (`11.11 ms` / Frame)**:
   - On legacy Z-Stack with an aged NV flash log at mesh depth D=150, every secured frame forces `NVOCMP` to walk flash pages: **`533,156` CPU cycles (`11.11 ms`) per NWK encrypt** and **`532,182` cycles (`11.09 ms`) per NWK decrypt**.
   - A single routed hop (`1` NWK decrypt + `1` NWK encrypt) consumes **`22.19 ms` of pure CPU time**, imposing a hard **`~45 routed packets/second` CPU ceiling** before any radio airtime or serial I/O is even counted.
   - Per-frame APS TX security (`APSME_LookupExtAddr` + TCLK NV reads) burns up to **`409,301` cycles (`8.53 ms`)** per frame across a 300-peer table.
2. **30.96 ms TCLK Lookup Miss Stalls During Device Joins & Interviews**:
   - When an unjoined or rejoining Zigbee 3.0 device contacts the Trust Center, `SSP_UpdateNwkKey` / TCLK lookup scans all 300 NV TCLK slots across flash pages. On legacy Z-Stack, a single TCLK lookup miss burns **`1,486,228` CPU cycles (`30.96 ms`)** with the scheduler blocked (and an oldest-slot hit at age 383 still burns **`620,396` cycles = `12.92 ms`**).
   - Under a 50-device join or post-outage rejoin burst, the coordinator stalls for **`1.70 seconds` (`33.9 ms` per device)**, causing host `SREQ` commands and BDB `Request-Key` timers to time out.
3. **163.9 ms Blocking `NVOCMP` Flash Compactions (Every 69 Writes)**:
   - Because stock Z-Stack appends a new flash record even when the 100-byte state record is byte-for-byte identical (**`100,714` cycles = `2.10 ms` per write**), it triggers a blocking flash page compaction on **write #69 (`163.9 ms` CPU stall with interrupts masked)**.
4. **3.00-Second CPU Lockups Under Broadcast & Replay Storms**:
   - In a 50-router broadcast echo storm (`1,000` echoes at D=150), legacy Z-Stack spends `143,865` cycles (`3.00 ms`) per echo (**`3.00 seconds` total CPU lockup**). Under a 1,000-frame replay flood, it spends `143,802` cycles (`3.00 ms`) per replay (**`2.99 seconds` total CPU freeze**), starving `NPITask` and dropping the `SRSP`.

---

## 3. Immediate Mitigations for Legacy / Stock Firmware Users

If you are still running legacy stock Z-Stack firmware and cannot immediately flash ZeroStack:

1. **Throttle Chatty Tuya / Metering Devices**: Configure reporting intervals (`min_rep_interval >= 10s`) on AC-powered Tuya TS011F / `_TZ3000_` power plugs that flood the coordinator with unsolicited `AREQ` attribute reports.
2. **Convert Large Unicast Loops to Zigbee Groups**: Avoid sending individual `AF_DataRequest` commands to 15+ lights simultaneously in Home Assistant automations; bind them to a single Zigbee Group (`AF_DataRequestExt` groupcast) so only one `SREQ` transits the UART.
3. **Verify Hardware `RTS/CTS` vs. `rtscts: false`**: Hardware RTS/CTS (`DIO10` / `DIO11`) is wired on **SLZB-06U**, **SLZB-06P7U**, and **SLZB-06P10**. On **SLZB-06**, **SLZB-06P7v1**, **SLZB-07P7**, **SLZB-07P10**, or TCP socket bridges (`tcp://...:6638`), set `rtscts: false` in `configuration.yaml`.

---

## 4. How SMLIGHT ZeroStack Permanently Eliminates This Fault

**[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** eliminates `SRSP` timeouts and interview stalls across **SLZB-06**, **SLZB-06U**, **SLZB-06P7** / **SLZB-06P7v1**, **SLZB-06P7U**, **SLZB-07P7**, **SLZB-06P10**, and **SLZB-07P10**:

| Operation / Stress Scenario (D=150..300) | Stock TI Z-Stack (`DWT->CYCCNT`) | SMLIGHT ZeroStack (`Release Zero 0.9.9-beta.1`) | Hardware-Measured Improvement |
| :--- | :--- | :--- | :--- |
| **Per-Frame NWK Encrypt (D=150)** | `533,156` cyc (`11.11 ms`) | **`3,968` cyc (`82.7 µs`, `0` flash reads)** | **134.4× faster** (O(1) constant time) |
| **Per-Frame NWK Decrypt (D=150)** | `532,182` cyc (`11.09 ms`) | **`3,824` cyc (`79.7 µs`, `0` flash reads)** | **139.2× faster** (O(1) constant time) |
| **Routed Hop Security Ceiling (D=150)** | `22.19 ms` (`~45 pkt/s` max) | **`162 µs` (`7,792` cyc, `>6,100 pkt/s` max)** | **135× higher routed throughput ceiling** |
| **TCLK Lookup Miss (300-entry table)** | `1,486,228` cyc (`30.96 ms`) | **`326` cyc (`6.8 µs`, `0` flash reads)** | **4,559× faster** (oldest-record hit: **1,915× faster**) |
| **50-Device Join / TCLK Miss Flood (D=300)** | `1.70 s` total stall (`33.9 ms`/dev) | **`0.22 s` total wire time (`4 µs` / `208` cyc on-CPU)** | **7.7× faster join-storm completion** |
| **Per-Frame APS TX Security (300 peers)** | Up to `409,301` cyc (`8.53 ms`) | **`5,410 – 5,424` cyc (`113 µs`)** | **Up to 75.5× faster** |
| **Host Serial `SYS_PING` Reply Path** | `4,921` cyc (`9,790` cyc total) | **`816` cyc (`4,618` cyc total)** | **83% faster reply path (53% fewer total cycles)** |
| **50-Router Broadcast Echo Storm (`1,000` echoes)** | `3.00 s` total CPU (`3.00 ms`/echo) | **`151.16 ms` total CPU (`147 µs` p50/echo)** | **19.9× lower total CPU (20.3× faster per echo)** |
| **Idempotent NVM State Write & Compaction** | `100,714` cyc; compacts at write #69 (`163.9 ms`) | **`3,770` cyc (`0 B` flash writes); compacts at #1,013 (`13.7 ms`)** | **26.7× faster write + 14.7× lower compaction rate** |
