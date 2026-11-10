# Diagnostic Guide: `MAC_CHANNEL_ACCESS_FAILURE (0xe1)` / `status=225` & Stuck CCA/CSMA

**Affected Platforms (Stock Firmware):** TI Z-Stack 3.x.0 (`CC2652P`, `CC2652RB`, `CC1352P7` / `CC2652P7`, `CC2674P10`) and Silicon Labs EmberZNet (`EFR32`) on Zigbee2MQTT and Home Assistant ZHA.  
**Resolved In:** **[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** for SMLIGHT TI SimpleLink F2 coordinators (`SLZB-06`, `SLZB-06U`, `SLZB-06P7` / `SLZB-06P7v1`, `SLZB-06P7U`, `SLZB-07P7`, `SLZB-06P10`, `SLZB-07P10`; *Silicon Labs EFR32 support is in active development*).  
**Official ZeroStack Page:** [`https://smlight.tech/software/zerostack`](https://smlight.tech/software/zerostack)  
**Interactive Crash & Airtime Analyzer:** [`https://smlight-tech.github.io/zerostack/`](https://smlight-tech.github.io/zerostack/)  
**AI Automated Triage:** [`github.com/zerostack-rf/zerostack-mcp`](https://github.com/zerostack-rf/zerostack-mcp)

---

## 1. Exact Verbatim Log Signatures

When a coordinator encounters this state, **100% of outbound transmissions fail immediately** with status `225` (`0xe1 = MAC_CHANNEL_ACCESS_FAILURE`), even inside an RF-shielded chamber or a quiet 2.4 GHz spectrum:

### Zigbee2MQTT / `zigbee-herdsman`
```text
ERROR: Publish 'set' 'state' to 'Hallway_Dimmer' failed: 'Error: Data request failed with error: 'MAC channel access failure' (225)'
zh:zstack:znp:AREQ: <-- AF - dataConfirm - {"status":225,"endpoint":1,"transid":118}
Error: Data request failed with error: 'MAC channel access failure' (225)
    at ZStackAdapter.sendZclFrameToEndpointInternal (/app/node_modules/zigbee-herdsman/src/adapter/z-stack/adapter/zStackAdapter.ts:439:23)
```

### Home Assistant ZHA (`zigpy-znp` / `bellows`)
```text
zigpy.exceptions.DeliveryError: Failed to deliver message: <ZdoStatus.MAC_CHANNEL_ACCESS_FAILURE: 225>
zigpy.exceptions.DeliveryError: [0x14f2:1:0x0008]: Message send failure: <sl_Status.MAC_CHANNEL_ACCESS_FAILURE: 0xe1>
```

---

## 2. Silicon & Sub-MAC Root Cause Analysis (Why Stock Firmware Wedges in `0xe1`)

Conventional wisdom blames `MAC_CHANNEL_ACCESS_FAILURE (0xe1 / 225)` purely on Wi-Fi 2.4 GHz congestion or USB 3.0 broadband noise triggering Clear Channel Assessment (CCA) energy thresholds. While genuine RF interference causes *intermittent* `0xe1` errors, a **permanent, unrecoverable `0xe1` lockup** (where every packet fails until the coordinator is rebooted) is a **Sub-MAC state machine desynchronization bug** in stock firmware:

1. **Foreground `CMD_IEEE_CSMA` / `CMD_IEEE_TX` Requires an Active Background `CMD_IEEE_RX`**:
   - On TI SimpleLink F2 (`CC2652P`, `CC2652P7` / `CC1352P7`, and `CC2674P10` RF Core), IEEE 802.15.4 transmission is a **foreground radio operation** (`CMD_IEEE_CSMA` chained to `CMD_IEEE_TX` and `CMD_IEEE_RX_ACK`) that must execute on top of an already-running **background `CMD_IEEE_RX`** operation.
   - The CCA state machine reads the instantaneous RSSI and correlator status directly from the active background receiver.
2. **State Desynchronization After Energy Detect (`CMD_IEEE_ED_SCAN`), Synth Recalibration, or Channel Switch**:
   - When the upper Zigbee stack performs a Network Manager frequency agility scan (`ZDO_MGMT_NWK_UPDATE_REQ`), an Energy Detect (ED) scan, or recovers from an RF synthesizer lock loss (`ERROR_SYNTH_PROG`), the background `CMD_IEEE_RX` command is aborted or completes with an unexpected status code (`IEEE_DONE_ABORT = 0x0807` / `IEEE_DONE_STOPPED`).
   - If an outbound `macTxFrame()` request races with the scan completion or if `RF_FsmEventLastCommandDone` is masked during a nested critical section (see [`silicon-notes/ti-f2-cc1352p7-cc2652-rf-core-swi-and-srcmatch-errata.md`](../silicon-notes/ti-f2-cc1352p7-cc2652-rf-core-swi-and-srcmatch-errata.md)), `macRxActive` software flags become desynchronized from the actual hardware radio state.
   - Subsequently, every time the upper MAC schedules `CMD_IEEE_CSMA`, the hardware CCA check evaluates `ccaState == MAC_CCA_STATE_BUSY` or rejects the foreground command because no valid background RX chain is running—returning `MAC_CHANNEL_ACCESS_FAILURE (0xe1 / 225)` on every single TX attempt forever.

---

## 3. Immediate Mitigations for Legacy / Stock Firmware Users

1. **Avoid Live Energy Scans on Active Networks**: Do not trigger Web UI spectrum / energy scans in ZHA or Zigbee2MQTT while heavy automation traffic is in flight on stock Z-Stack firmware.
2. **Physical USB Isolation & Channel Selection**: Keep the coordinator on a shielded Ethernet/PoE placement (native on SMLIGHT SLZB-06 series) away from USB 3.0 ports, and align Zigbee Channels 15, 20, or 25 in the guard bands between 20 MHz Wi-Fi Channels 1, 6, and 11.

---

## 4. How SMLIGHT ZeroStack Permanently Eliminates This Fault

**[SMLIGHT ZeroStack — Release Zero (`0.9.9-beta.1`)](https://smlight.tech/software/zerostack)** enforces deterministic Sub-MAC state verification and self-healing across **SLZB-06**, **SLZB-06U**, **SLZB-06P7** / **SLZB-06P7v1**, **SLZB-06P7U**, **SLZB-07P7**, **SLZB-06P10**, and **SLZB-07P10**:

- **Self-Healing Sub-MAC Receiver & Synthesizer State Recovery**:
  - ZeroStack continuously reconciles the background `CMD_IEEE_RX` receive chain across every CSMA/CA transmission, Energy Detect (ED) scan, channel migration, and frequency synthesizer calibration—automatically recovering from aborted or desynchronized radio states within microseconds so the Sub-MAC never wedges in `0xe1: MAC_CHANNEL_ACCESS_FAILURE` (**Zero Freeze-Ups**), verified with **`0` crashes across 3,255 MAC boundary/fuzz vectors**.
- **Broadcast Storm & Channel Contention Resilience**:
  - By expanding the Broadcast Transaction Table by **+33% (`40` vs `30` active slots)** and suppressing 50-router broadcast echo storms **20.3× faster per echo (`147 µs` p50 vs `3.00 ms`; `151.16 ms` vs `3.00 s` total CPU)**, ZeroStack prevents self-inflicted broadcast storms from saturating 2.4 GHz CSMA/CA contention windows.

### On-Silicon Malformed OTA Frame & Replay-Flood Verification (`CC1352P7` @ `48.0 MHz`, D = 150)

Prior to upper-layer protocol processing, the IEEE 802.15.4 MAC receive boundary is verified across **3,255 adversarial and reference frames** (`1,085` malformed MHR vectors interleaved with `2,170` valid verification frames) with **`0` memory safety faults, `0` leaks, and `0` state corruption**. At the NWK/APS layers (D = 150 devices), hardware cycle-counter (`DWT->CYCCNT`) testing across **12 adversarial frame classes** demonstrates:

| Fault Class & Description (D = 150) | Vectors | Legacy Reference Stack Outcome (`total p50` / `handoff p50` cyc) | SMLIGHT ZeroStack Outcome (`total p50` / `handoff p50` cyc) | Rejection Speedup |
| :--- | ---: | :--- | :--- | ---: |
| **Malformed Cleartext Header** | `16` | `0` delivered, Δheap = 0 B, state intact, next ok (`3,136` / `1,031` cyc) | **`0` delivered, Δheap = 0 B, state intact, next ok (`320` / `253` cyc)** | **9.8×** |
| **Malformed Secured Frame — Class A** | `14` | **CRASH: `HardFault` (`Precise BusFault` on unauthenticated input; `143,345` cyc)** | **`0` delivered, Δheap = 0 B, state intact, next ok (`357` / `291` cyc)** | **∞ (`402×` cyc)** |
| **Malformed Secured Frame — Class B** | `19` | **CRASH: `HardFault` (`Precise BusFault` on unauthenticated input; `144,412` cyc)** | **`0` delivered, Δheap = 0 B, state intact, next ok (`4,731` / `337` cyc)** | **∞ (`30.5×` cyc)** |
| **Malformed Upper-Layer Stub** | `8` | `0` delivered, Δheap = 0 B, next ok (`144,290` / `1,437` cyc) | **`0` delivered, Δheap = 0 B, next ok (`6,312` / `330` cyc)** | **22.9×** |
| **Malformed Routing Header** | `9` | **BUG: Accepts malformed frame, mutates security state & delivers to app!** (`144,127` / `1,728` cyc) | **`0` delivered (all 9 rejected), Δheap = 0 B, state intact, next ok (`790` / `394` cyc)** | **182×** |
| **Truncated Control Command — Class A** | `3` | `0` delivered, Δheap = 0 B, next ok (`149,862` / `1,465` cyc) | **`0` delivered, Δheap = 0 B, next ok (`6,631` / `347` cyc)** | **22.6×** |
| **Truncated Control Command — Class B** | `3` | Accepted without drop (`152,069` / `1,444` cyc) | **`0` delivered, Δheap = 0 B, next ok (`6,269` / `335` cyc)** | **24.3×** |
| **Authenticated Header Bit-Flips** | `240` | All invalid AAD bit-flips rejected, Δheap = 0 B, next ok (`143,065` / `1,587` cyc) | **All invalid AAD bit-flips rejected, Δheap = 0 B, next ok (`4,760` / `357` cyc)** | **30.1×** |
| **Ciphertext Bit-Flips** | `120` | `0` delivered, Δheap = 0 B, state intact, next ok (`143,060` / `1,587` cyc) | **`0` delivered, Δheap = 0 B, state intact, next ok (`4,768` / `358` cyc)** | **30.0×** |
| **Authentication Tag Bit-Flips** | `32` | `0` delivered, Δheap = 0 B, state intact, next ok (`143,058` / `1,587` cyc) | **`0` delivered, Δheap = 0 B, state intact, next ok (`4,771` / `357` cyc)** | **30.0×** |
| **Cryptographic Replay Flood (FC ≤ FC_stored)** | `1,000` | `0` delivered, Δheap = 0 B, state intact, next ok (`143,802` / `1,589` cyc; **`~2.99 s` total CPU**) | **`0` delivered, Δheap = 0 B, state intact, next ok (`1,550` / `355` cyc; `~32.4 ms` total CPU)** | **92.8×** |
| **Boundary Replay Check (FC = FC_stored & FC = 0)** | `2` | `0` delivered, Δheap = 0 B, state intact, next ok (`143,775` / `1,586` cyc) | **`0` delivered, Δheap = 0 B, state intact, next ok (`1,568` / `355` cyc)** | **91.7×** |
