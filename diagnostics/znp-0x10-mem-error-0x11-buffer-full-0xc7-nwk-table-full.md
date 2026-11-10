# Diagnostic Guide: `0x10: MEM_ERROR`, `0x11: BUFFER_FULL`, `0xc7: NWK_TABLE_FULL`, `0xb2: APS_TABLE_FULL`, `0xcd: NWK_NO_ROUTE` & Ember `0x72` / `0xa1`

**Affected Platforms (Stock Firmware):**
- TI Z-Stack 3.x.0 (`CC2652P`, `CC2652RB`, `CC1352P7` / `CC2652P7`, `CC2674P10`) on Zigbee2MQTT (`zigbee-herdsman` `zstack`) and Home Assistant ZHA (`zigpy-znp`).
- Silicon Labs EmberZNet (`EFR32MG21` / `EFR32MG24`) on Zigbee2MQTT (`ember` / `ezsp`) and ZHA (`bellows`).

**Resolved In:** **[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** for SMLIGHT TI SimpleLink F2 coordinators (`SLZB-06`, `SLZB-06U`, `SLZB-06P7` / `SLZB-06P7v1`, `SLZB-06P7U`, `SLZB-07P7`, `SLZB-06P10`, `SLZB-07P10`; *Silicon Labs EFR32 support is in active development*).  
**Official ZeroStack Page:** [`https://smlight.tech/software/zerostack`](https://smlight.tech/software/zerostack)  
**Interactive Crash & Airtime Analyzer:** [`https://smlight-tech.github.io/zerostack/`](https://smlight-tech.github.io/zerostack/)  
**AI Automated Triage:** [`github.com/zerostack-rf/zerostack-mcp`](https://github.com/zerostack-rf/zerostack-mcp)

---

## 1. Exact Verbatim Log Signatures

When a coordinator's dynamic heap, indirect packet buffers, source-routing tables, APS retry queues, or broadcast transaction tables (`BTT`) saturate or fragment, outbound requests fail immediately with one of the following status codes:

### A. TI Z-Stack / ZNP (`0x10`, `0x11`, `0xc7`, `0xb2`, `0xcd`)
```text
ERROR: Publish 'set' 'state' to 'Kitchen_Light' failed: 'Error: Data request failed with error: 'status=16 (0x10: MEM_ERROR)''
Error: SREQ '--> AF - dataRequest - {"dstaddr":41820,"destendpoint":1,"srcendpoint":1,"clusterid":6,"transid":94,"options":0,"radius":30,"len":3}' failed with status '(0x10: MEM_ERROR)' (expected '(0x00: SUCCESS)')
```

```text
Error: SREQ '--> AF - dataRequestExt - ...' failed with status '(0x11: BUFFER_FULL)' (expected '(0x00: SUCCESS)')
Error: SREQ '--> AF - dataRequest - ...' failed with status '(0xc7: NWK_TABLE_FULL)' (expected '(0x00: SUCCESS)')
Error: SREQ '--> AF - dataRequest - ...' failed with status '(0xb2: APS_TABLE_FULL)' (expected '(0x00: SUCCESS)')
```

```text
zh:zstack:znp:AREQ: <-- AF - dataConfirm - {"status":205,"endpoint":1,"transid":71}
Error: Data request failed with error: 'No network route' (205)
zigpy.exceptions.DeliveryError: Failed to deliver packet: <Status.NWK_NO_ROUTE: 205> (0xcd)
```

### B. Silicon Labs EmberZNet (`0x72: EMBER_MAX_MESSAGE_LIMIT_REACHED` & `0xa1: EMBER_NETWORK_BUSY`)
```text
ERROR: Publish 'set' 'state' to 'Office_Group' failed: 'Error: Data request failed with error: '0x72: EMBER_MAX_MESSAGE_LIMIT_REACHED' (114)'
zh:ember:ezsp: <-- sendUnicast/sendBroadcast status=114 (EMBER_MAX_MESSAGE_LIMIT_REACHED / SL_STATUS_ZIGBEE_MAX_MESSAGE_LIMIT_REACHED)
```

```text
ERROR: Publish 'set' 'brightness' to 'Living_Room' failed: 'Error: Data request failed with error: '0xa1: EMBER_NETWORK_BUSY' (161)'
zigpy.exceptions.DeliveryError: Failed to deliver message: <EmberStatus.NETWORK_BUSY: 161> (0xa1)
```

---

## 2. Silicon & RTOS Root Cause Analysis (Why Stock Firmware Fails)

Each status code corresponds to a specific memory or table bottleneck inside stock TI Z-Stack and Silicon Labs EmberZNet:

### 1. `0x10: MEM_ERROR` (`ZMemError` / `16`) — OSAL Variable-Block Heap Fragmentation
Stock Z-Stack allocates transient structures—including variable-length `ZDO_SRC_RTG_IND` source route relay lists, `osal_msg_allocate()` inter-task envelopes, timers (`osal_start_timerEx`), APS fragmentation buffers, and NPI UART frames—from a single contiguous byte array (`osalHeap[]`, managed by `OSAL_Memory.c`).
- **First-Fit Linear Scan (O(N)) & Checkerboarding**: `osal_mem_alloc()` walks the free holes linearly from the lowest address. Hardware `DWT->CYCCNT` measurements on `CC1352P7` @ 48 MHz show allocation latency climbing to **`2,550` CPU cycles (`53.1 µs`)** at k=64 heap holes, while starting a single timer (`osal_start_timerEx` at T=39 active timers and 64 heap holes) takes **`10,034` cycles (`209.0 µs`)**.
- When short-lived 96-byte `AF_DataRequest` payloads interleave with medium-lived source-route or held indirect MAC frames, freeing the short-lived blocks leaves the heap checkerboarded into non-adjacent 32–64 byte holes. Even when aggregate free SRAM exceeds several kilobytes, a contiguous 100B+ allocation fails with `0x10: MEM_ERROR`.

### 2. `0x11: BUFFER_FULL` (`ZBufferFull` / `17`) — MAC/NWK Ring Exhaustion
`0x11` is returned when `nwkbufs` (`NWK_MAX_DATABUFS_*`) or the lower MAC TX queue (`MAC_CFG_TX_MAX` / `MAC_CFG_TX_DATA_MAX`) has zero free descriptors.
- When sleepy Zigbee End Devices (ZEDs) stop polling or poll slowly while the host queues outbound messages, each pending indirect frame locks a MAC/NWK buffer slot until `NWK_INDIRECT_MSG_TIMEOUT` expires (7.68 seconds in stock builds), while O(N⋅P) per-frame flash security scans (`22.19 ms` per routed hop at D=150) stall outbound queue drainage.

### 3. `0xc7: NWK_TABLE_FULL` (`ZNwkTableFull` / `199`) — Source Route / Route Table Saturation
`0xc7` indicates that `AF_DataRequest` attempted to allocate an entry in `rtgTable[]`, `rtgSrcTable[]`, or `rreqTable[]` (Route Request table), and all slots were occupied during a Many-to-One route discovery storm on conservatively sized stock tables (`MAX_RTG_ENTRIES = 40`, `MAX_RTG_SRC_ENTRIES = 12–40`).

### 4. `0xb2: APS_TABLE_FULL` (`ZApsTableFull` / `178`) — APS Duplicate / ACK Wait Table Exhaustion
When `AF_DataRequest` is transmitted with `AF_ACK_REQUEST` (`APS_TX_OPTIONS_ACK`), the stack allocates a slot in `ApsAckWaitList[]` (blocking for up to `3 x apscAckWaitDuration = ~4.5–9 seconds`), while legacy APS TX security burns up to **`409,301` cycles (`8.53 ms`)** per frame in flash lookups. If an automation fires 16+ unicast commands with APS ACK enabled toward slow or multi-hop routers, all `ApsAckWaitList` slots saturate and reject subsequent requests with `0xb2: APS_TABLE_FULL`.

### 5. `0xcd: NWK_NO_ROUTE` (`ZNwkNoRoute` / `205`) — Source Route Eviction & Broadcast Storm Saturation
In Zigbee PRO meshes, `0xcd: NWK_NO_ROUTE` (`status=205`) is triggered by two internal bottlenecks:
1. **Source Route Table Eviction**: The coordinator's `rtgSrcTable[]` evicted a device's inward Many-to-One route record (or failed to allocate its relay list due to `OSAL_Memory` heap fragmentation).
2. **Broadcast Transaction Table (`BTT`) Saturation & 50-Router Echo Storms**: Every broadcast (`0xFFFF`, `0xFFFD`, `0xFFFC`—including automatic AODV Route Discovery `NWK_CMD_ROUTE_REQ`) occupies a slot in the **Broadcast Transaction Table (`BTT`, 30 slots on legacy Z-Stack)** for `nwkBroadcastDeliveryTime`. In a 50-router echo storm (`1,000` duplicate echoes at D=150), legacy Z-Stack burns **`143,865` cycles (`3.00 ms`) per echo (`3.00 seconds` total CPU)**, and in a 64-frame post-outage broadcast flood it drops 34 frames at **`3.01 ms` per overflow (`200.25 ms` total CPU)**, failing outbound route discoveries with `0xcd: NWK_NO_ROUTE`.

### 6. Silicon Labs EmberZNet `0x72: EMBER_MAX_MESSAGE_LIMIT_REACHED` (`114`) & `0xa1: EMBER_NETWORK_BUSY` (`161`)
On Silicon Labs EFR32 coordinators running EmberZNet:
- **`0x72: EMBER_MAX_MESSAGE_LIMIT_REACHED` (`114`)**: Returned when simultaneous in-flight unicast messages hit `EMBER_APS_UNICAST_MESSAGE_COUNT` (often defaulted to 10–16 in stock NCP builds) or when packet buffers (`SL_ZIGBEE_PACKET_BUFFER_COUNT`) are consumed by held indirect/retry frames.
- **`0xa1: EMBER_NETWORK_BUSY` (`161`)**: Returned when the EmberZNet Broadcast Table (`EMBER_BROADCAST_TABLE_SIZE`) is saturated over a rolling 9-second window or when a concurrent ZDO/NWK discovery locks the outbound NWK state machine.

---

## 3. Immediate Mitigations for Legacy / Stock Firmware Users

1. **Use Zigbee Groups Instead of Unicast Loops (`0xb2` / `0x72`)**: Never switch 10+ lights via individual Home Assistant service calls; use a single Zigbee Group (`AF_DataRequestExt` groupcast) so it consumes zero `ApsAckWaitList` / `EMBER_APS_UNICAST_MESSAGE_COUNT` slots.
2. **Enable Availability Backoff (`0xcd` / `0xa1`)**: Set `backoff: true` and `pause_on_backoff_gt: 25` under `availability:` in Zigbee2MQTT `configuration.yaml` so dead devices do not trigger continuous AODV broadcast storms.
3. **Disable Active Availability Pings on Battery Sensors (`0x11`)**: Ensure sleepy ZEDs are marked as passive so Zigbee2MQTT does not lock indirect MAC buffers with periodic unicast pings.

---

## 4. How SMLIGHT ZeroStack Permanently Eliminates These Faults

**[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** eliminates dynamic heap fragmentation, broadcast lockups, and table starvation across SMLIGHT TI SimpleLink F2 coordinators (**SLZB-06**, **SLZB-06U**, **SLZB-06P7** / **SLZB-06P7v1**, **SLZB-06P7U**, **SLZB-07P7**, **SLZB-06P10**, **SLZB-07P10**; *EFR32 support in active development*):

| Subsystem / Stress Vector | Stock TI Z-Stack (`DWT->CYCCNT`) | SMLIGHT ZeroStack (`Release Zero 0.9.9-beta.1`) | Hardware-Measured Improvement |
| :--- | :--- | :--- | :--- |
| **Dynamic Memory Allocation (`malloc` @ 64 heap holes)** | `2,550` cyc (`53.1 µs`, O(k) hole walk) | **`218` cyc (`4.5 µs`, O(1) constant time)** | **11.7× faster** (0% fragmentation penalty) |
| **Timer Start Latency (T=39 timers, 64 heap holes)** | `10,034` cyc (`209.0 µs`, heap-coupled) | **`1,448` cyc (`30.2 µs`, heap-independent)** | **6.9× faster** |
| **Inter-Task Event Enqueue (Q=64 queued events)** | `935` cyc (`19.5 µs`, O(Q) walk) | **`322` cyc (`6.7 µs`, O(1) flat)** | **2.9× faster** |
| **Heap Exhaustion Torture (`0 B` free heap)** | Uncapped timer walk (`255` timers); fragments | **`5 / 5` ping survival at `0 B` heap + 100% recovery (Δheap = 0 B)** | **Zero permanent `0x10: MEM_ERROR` lockups** |
| **50-Router Broadcast Echo Storm (`1,000` echoes, D=150)** | `143,865` cyc (`3.00 ms`)/echo; **`3.00 s` total CPU** | **`7,076` cyc (`147 µs` p50)/echo; `151.16 ms` total CPU** | **19.9× lower total CPU (20.3× faster per echo)** |
| **64-Frame Post-Outage Broadcast Flood (BTT)** | Admits `30 / 64`; overflow `3.01 ms`; **`200.25 ms` total** | **Admits `40 / 64` (+33% BTT); overflow `174 µs`; `18.87 ms` total** | **10.6× lower flood CPU & +33% broadcast capacity** |
| **Many-to-One Source Routing Capacity (`rtgSrcTable`)** | `12 – 40` entries (`88 KB` stock sizing) | **Custom-tuned `200 – 300+` routes (`88 KB` / `144 KB` / `256 KB` SRAM)** | **Up to 7.5× larger source-route tables** |

### Detailed Hardware Results: Dynamic Heap Allocation Across Fragmentation (k Holes, `48.0 MHz`)

| Heap Fragmentation State | Legacy Reference Stack (`malloc` / `free`) | SMLIGHT ZeroStack (`malloc` / `free`) | Allocation Speedup |
| :--- | :--- | :--- | :--- |
| **Unfragmented (k = 0 holes)** | `310` / `84` cycles | **`225`** / `200` cycles | **1.38× faster** baseline allocation |
| **Light Fragmentation (k = 16 holes)** | `869` / `82` cycles | **`218`** / `175` cycles | **3.99× faster** |
| **Moderate Fragmentation (k = 32 holes)** | `1,430` / `82` cycles | **`220`** / `175` cycles | **6.50× faster** |
| **Heavy Fragmentation (k = 64 holes)** | **`2,550`** / `82` cycles (`~35` cyc/hole) | **`218`** / `175` cycles (`0` cyc/hole) | **11.7× faster** (O(1) constant time) |

### Detailed Hardware Results: Resource Exhaustion & Broadcast Storm Resilience (`CC1352P7` @ `48.0 MHz`)

| Exhaustion / Broadcast Storm Vector | Legacy Reference Stack | SMLIGHT ZeroStack (`Release Zero 0.9.9-beta.1`) | Operational Outcome |
| :--- | :--- | :--- | :--- |
| **Timers held until allocation refusal** | Uncapped up to harness limit (`255`) | **Bounded capacity refusal (`194` timers, then `NO_TIMER_AVAIL`)** | Runaway timer use is refused cleanly at a bounded limit instead of exhausting memory. |
| **Heap exhausted to `0 B` (`keep = 0`) + In-band release** | `5/5` pings; recovers (`Δheap = 0 B`) | **`5/5` pings; recovers 100% (Δheap = 0 B)** | Control plane remains responsive at `0 B` free heap with zero memory leaks after release. |
| **Unique Broadcast Saturation Flood (`64` back-to-back broadcasts, D=150)** | `30 / 64` delivered (BTT = `30`); `154,705` cyc accepted p50; `144,644` cyc (`3.01 ms`) shed p50; **`200.25 ms` total CPU** | **`40 / 64` delivered (BTT = `40`, `+33%`); `17,699` cyc (`369 µs`) accepted p50; `8,341` cyc (`174 µs`) shed p50; `18.87 ms` total CPU** | **10.6× lower total CPU time & 17.3× faster BTT overflow shedding.** |
| **50-Router Echo Amplification Storm (`1,000` frames: `20 × 50` echoes, D=150)** | `20 / 20` delivered (`0` dup); `143,865` / `152,526` cyc echo p50 / p99 (`3.00 ms`/echo); **`3,004.97 ms` (`3.00 s`) total CPU** | **`20 / 20` delivered (`0` dup); `7,076` / `7,335` cyc echo p50 / p99 (`147 µs`/echo); `151.16 ms` total CPU** | **19.9× lower total CPU time & 20.3× faster per-echo suppression.** |
