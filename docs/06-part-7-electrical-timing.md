# Sony MFD-51W Drive Interface Protocol — Part 7: Electrical Requirements and Timing

**Document:** 06 — Part 7  
**Protocol section:** **15 — Electrical requirements and timing**  
**Status:** Source-grounded interface requirements and agreed design constraints; selected circuits, STM32 peripherals and prototype measurements remain **[OPEN]**  
**Target:** Original Macintosh Plus M0001AP **internal 20-pin** connector, emulating an internal Sony **MFD-51W 800K** mechanism  
**Date:** 11 November 2026

> **Document sequence:** [Document 02 — 20-pin connector](02-macintosh-20-pin-interface.md) is the authoritative host-specific pin assignment; [Part 1 / §§1–9](06-part-1-drive-interface.md) specifies selector/status/control, TACH, steps, READY and eject; [Part 2 / §10](06-part-2-serial-gcr-read.md) specifies GCR RD; [Part 3 / §11](06-part-3-media-architecture.md) specifies buffering and storage; [Part 4 / §12](06-part-4-serial-gcr-write.md) specifies WR capture; [Part 5 / §13](06-part-5-format-geometry-400k.md) specifies ROM-compatible tracks; [Part 6 / §14](06-part-6-error-handling-standard-media.md) specifies standard-media faults. **This section addresses only host-facing electrical characteristics, waveform constraints, safe interfacing and measurable acceptance.** [Part 8 / §16](06-part-1-drive-interface.md#planned-section-16-stm32-implementation-architecture) will choose the MCU/peripherals, buffers, supplies and firmware arrangement.

**Source keys:** **[APPLE]** Macintosh Plus connector/power in Apple's *Guide to the Macintosh Family Hardware*, second edition, chapter 9, table 9-3; **[SONY]** Apple/Sony **699-0452-A**, *800K Double-Sided Drive Engineering Requirements Specification* (September 1986), especially §§2.3, 3.1–3.6, the cited drawing sheets; **[ROM]** Macintosh Plus Sony driver; **[DECISION]** approved emulator requirement; **[PROPOSED]** suggested circuit/engineering practice; **[OPEN]** not yet measured or selected. **Do not transfer 400K or external-drive pin assignments into the internal MFD-51W design.**

Primary sources:

- [Apple: Guide to Macintosh Family Hardware, second edition (1990), chapter 9, Macintosh Plus internal Table 9-3 (printed p. 335)](https://vintageapple.org/inside_o/pdf/Guide_to_Macintosh_Family_Hardware_2nd_Edition_1990.pdf).
- [Apple/Sony 699-0452-A 800K drive engineering specification (1986), 47 sheets](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_specs/669-0452-A_800K_Double-Sided_ERS_Sep86.pdf).
- [Document 02: researched M0001AP 20-pin pinout](02-macintosh-20-pin-interface.md).
- [Plus ROM source: Sony driver](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s).

## Contents

- [15.1 Scope and electrical boundary](#151-scope-and-electrical-boundary)
- [15.2 Macintosh Plus internal 20-pin pin assignment](#152-macintosh-plus-internal-20-pin-pin-assignment)
- [15.3 DC logic levels, loading and RD output drive](#153-dc-logic-levels-loading-and-rd-output-drive)
- [15.4 RD enable, high impedance and status mux timing](#154-rd-enable-high-impedance-and-status-mux-timing)
- [15.5 Serial RD waveform and GCR timing](#155-serial-rd-waveform-and-gcr-timing)
- [15.6 WRDATA capture, write gate and RD handoff](#156-wrdata-capture-write-gate-and-rd-handoff)
- [15.7 Command selector and LSTRB timing](#157-command-selector-and-lstrb-timing)
- [15.8 STEP, direction, TK0 and READY timing](#158-step-direction-tk0-and-ready-timing)
- [15.9 TACH timing and motor-status behavior](#159-tach-timing-and-motor-status-behavior)
- [15.10 Media presence, eject and power-fail timing](#1510-media-presence-eject-and-power-fail-timing)
- [15.11 Power rails, protection and mixed-voltage architecture](#1511-power-rails-protection-and-mixed-voltage-architecture)
- [15.12 Interface implementation boundaries and timing priority](#1512-interface-implementation-boundaries-and-timing-priority)
- [15.13 Bench verification and acceptance matrix](#1513-bench-verification-and-acceptance-matrix)
- [15.14 Decisions, discrepancies and open implementation work](#1514-decisions-discrepancies-and-open-implementation-work)

---

## 15. Electrical requirements and timing

### 15.1 Scope and electrical boundary

**[DECISION]** Replace the **drive mechanism's host-visible interface only**; the Macintosh retains its actual IWM, ROM, MC68000 and VIA. No high-level floppy commands, fabricated controller, separate track-zero/index pin, byte-wide GCR link, or external pause/ACK is inserted.

The only normal **drive-to-host signal** on the connector is **RD (pin 16)**. Its value is either a selected **status/control-readback level**, a real-time **serial GCR RD pulse stream**, or **high impedance when disabled**. This multiplexing and its electrical timing are critical.

Host-to-drive lines are input signals; their level changes must not be misinterpreted as write data unless the correct gate/selection state is active. The ROM can rapidly change selector, status and serial modes.

**[DECISION]** The absence of physical mechanics eliminates *simulated* acoustic, seek, spin-up and rotational delays, **not** the electrical minimum/maximum constraints at the 20-pin connector. Do not insert arbitrary latency into command or RD response to imitate a moving head.

**[DECISION]** This section specifies an **internal MFD-51W interface only**. The external Macintosh DB-19 interface and Sony MFD-51W-03 / MFD-51W-10 variant connections remain separate future interfaces.

### 15.2 Macintosh Plus internal 20-pin pin assignment

**[APPLE / DECISION]** Use the **Macintosh Plus-specific** Table 9-3, not later Macintosh pinouts or generic online 20-pin diagrams:

| Pin | Plus signal | Direction seen by emulator | Function / treatment |
|---:|---|---|---|
| 1 | GND | Ground | Power/signal return |
| 2 | PH0 | Input | Sony CA0 / selector bit |
| 3 | GND | Ground | Power/signal return |
| 4 | PH1 | Input | Sony CA1 / selector bit |
| 5 | GND | Ground | Power/signal return |
| 6 | PH2 | Input | Sony CA2 / control value |
| 7 | GND | Ground | Power/signal return |
| 8 | PH3 | Input | Sony LSTRB / command strobe |
| **9** | **−12 V** | **Unconnected** | **Leave electrically disconnected** |
| 10 | /WRREQ | Input | Sony /WRTGATE function, active low |
| 11 | +5 V | Supply available | Original drive supply; regulator input TBD |
| 12 | SEL | Input | Status selector / data head selection context |
| 13 | +12 V | Supply available | Original drive supply; consumption TBD |
| 14 | /ENBL.INT | Input | Drive enable, active low |
| 15 | +12 V | Supply available | Original drive supply; consumption TBD |
| **16** | **RD** | **Tri-state output** | Only host-facing drive data/status line |
| 17 | +12 V | Supply available | Original drive supply; consumption TBD |
| **18** | **WR** | **Input** | Encoded write-data level transitions, both edges |
| 19 | +12 V | Supply available | Original drive supply; consumption TBD |
| **20** | **PWM** | **Unconnected** | **Leave electrically disconnected** for MFD-51W |

**[DECISION — non-negotiable]** Pins **9 and 20 remain electrically disconnected**, without pull-up, pull-down, ESD path to MCU rails, GPIO, regulator or ground. The host presents **−12 V** at pin 9; confusing it with +5 V or ground can damage hardware. Pin 20 is a host PWM output for legacy 400K speed control but is **not used** by the internal MFD-51W emulator.

**[OPEN]** Verify the exact physical 20-pin harness orientation, keying, connector numbering, continuity and host voltages **before first powered attachment**. A schematic pin table alone is not a safe wiring diagram. Document 02 owns the host pinout, this part owns electrical acceptance.

### 15.3 DC logic levels, loading and RD output drive

**[SONY]** The 1986 699-0452-A **sheet 22**, §3.3 specifies drive-interface levels (voltages relative to signal ground):

| Parameter | 699-0452-A limit | Meaning |
|---|---:|---|
| Drive **input VIH** | **≥2.2 V** | High guaranteed recognized at original drive input |
| Drive **input VIL** | **≤0.8 V** | Low guaranteed recognized at original drive input |
| Non-WR host-input **IIH** | **≤0.01 mA** | Original input high leakage/load |
| Non-WR host-input **IIL** | **−0.61 mA** at specification condition | Original low-level sink/input load |
| WRDATA / WRTGATE **IIH** | **−0.9 mA** as specified | Original input-loading characteristic; not a new source current |
| WRDATA / WRTGATE **IIL** | **−1.5 mA** as specified | Original input-loading characteristic |
| Drive RD **VOH** | **≥2.4 V at IOH = −1.0 mA** | Original drive output-high guarantee |
| Drive RD **VOL** | **≤0.5 V at IOL = 8.0 mA** | Original drive output-low guarantee |

**[SONY]** WRDATA and WRTGATE on the original drive **include 3.3-kΩ pull-ups to +5 V**. This is an **original-load characteristic**, not evidence that the Macintosh host has such pull-ups. Reproducing appropriate drive-side loading matters; avoid silently adding multiple aggressive pull-ups or assuming the STM32 input alone duplicates this loading.

**[DECISION]** The selected emulator electronics must meet **the host-facing input thresholds and RD output drive** and operate safely against the host's 5 V-era signals. **3.3 V MCU GPIO is not automatically an acceptable 5 V logic output or 5 V-tolerant input**. VOH at the original Sony pin is a *minimum guaranteed level* under load, **not** an instruction to make an arbitrary 3.3 V RD driver without checking the Plus input and pull-up/circuit.

**[PROPOSED]** Use an appropriate **5 V-tolerant input-buffer/translator** for PH0–PH3, SEL, /ENBL, /WRREQ and WR, and a **low-delay 5 V-compatible tri-state output driver** for RD, with separate MCU-side 3.3 V logic. Verify the chosen parts' input capacitance, threshold/noise margins, output current, propagation delay, supply sequencing and power-off protection. Do not select a specific logic IC yet.

**[OPEN]** Determine the Plus motherboard's actual RD load/pull-up and rise/fall shapes, host output drive, cable capacitance, required pull-ups/terminations and worst-case mixed-voltage noise margins before completing the electrical schematic.

### 15.4 RD enable, high impedance and status mux timing

**[SONY]** 699-0452-A **sheet 23**, §3.4.1.1, shows:

| RD event | Maximum delay |
|---|---:|
| /ENBL becomes active (low) → selected RD valid | **0.5 µs** (T1) |
| CA0/CA1/CA2/SEL selector changes → new RD selected level | **0.5 µs** (T2) |
| /ENBL becomes inactive (high) → RD becomes **high impedance** | **0.5 µs** (T3) |

**[DECISION]** RD **must not remain actively driven** when the internal drive is disabled. Driving a logic 1 in place of high impedance is noncompliant and risks contention with another drive or source. Independent output-enable gating should ensure RD becomes high impedance even if firmware is stalled or the STM32 is resetting.

**[DECISION]** The status selector is a **combinational/fast-mux response** from the host's viewpoint; a slow periodic software poll of CA/SEL is not adequate to claim a ≤0.5-µs response. The selected status value must be correctly latched/generated according to Part 1, and selecting a status must **not** erroneously interpret an LSTRB command.

**[DECISION]** The read-data head and status selection share the **same RD output path**; switching between them must not produce a false extra pulse, bus fight or stale response. When enabled, only the selected stream/level may drive RD.

**[PROPOSED]** A fast hardware mux / small combinational logic / suitable STM32 timer+buffer-enable arrangement is plausible; actual selection is deferred to Section 16. Measure both input-to-output propagation **and RD tri-state turn-off** with a scope/logic analyzer, including reset, enable glitches and active serial output.

### 15.5 Serial RD waveform and GCR timing

**[SONY]** 699-0452-A **sheet 24**, §3.4.1.2, specifies the RDDATA waveform:

| Parameter | Requirement | Engineering consequence |
|---|---:|---|
| RD pulse width T4 | **0.3–0.8 µs** | Generate deterministic width on each recorded 1 cell |
| RD pulse separation T5 | **2, 4 or 6 µs nominal** | Encoded GCR transition intervals, not arbitrary busy-loop pulse spacing |
| /READY → valid RDDATA T1 | **≤0.5 µs** | Immediate valid-data selection requirement |
| SEL/data-source change T2 | **≤100 µs** | Head/data change; separate from 0.5-µs **status** selection |
| Gate release → restored RDDATA T3 | **≤620 µs** | Resume real serial data without flash-read delays |
| SEL → write-gate separation T6 | **≥100 µs** | Host-input sequence constraint; accept valid host timing |

**[SONY]** 699-0452-A **sheet 5**, §2.3.2, gives **489.6 kbit/s ±0.1%** and describes **~1.89–6.36 µs** transition separations across tolerance/recording conditions. The 2/4/6-µs values in the pulse drawing are rounded nominal intervals. **[DECISION]** The project's baseline is **489,600 bit cells/s**, one cell ≈ **2.042484 µs**, subject to the already planned logic-analyser confirmation and one global calibration. Actual GCR intervals follow the serialized one/zero pattern, normally 1, 2 or 3 cells between recorded transitions, **not a fixed 2-µs pulse train**.

**[DECISION]** A logical **1 cell generates an RD pulse** and a **0 cell generates no pulse**; the Sony/IWM bitstream must receive exact ROM-compatible SYNC6, headers, encoded 703-byte fields, gaps and seam (Parts 2, 3 and 5). Treat those as **serial cell patterns**, not SPI bytes written to RD directly.

**[DECISION]** RD timing remains valid during asynchronous flash erase/program, cache transitions and control/status changes. Hardware-timed pulse generation, or a demonstrably equivalent deterministic implementation, is required; CPU interrupt-driven per-transition output is not accepted without worst-case measured margin.

**[OPEN]** Confirm RD polarity, pulse shape/edge convention, rising/falling skew, jitter budget and IWM input tolerance against a real Plus with the selected driver. The **published 0.3–0.8 µs pulse-width band and source-switch constraints** are acceptance requirements; a particular pulse width inside that band remains a circuit choice.

### 15.6 WRDATA capture, write gate and RD handoff

**[SONY]** 699-0452-A **sheet 21**, §3.2.6: **each level transition** of WRTDATA represents a recorded flux transition, so **both rising and falling edges matter**. This is **not** an RD-style narrow-pulse input. **[ROM]** The IWM emits WR transitions while §/WRREQ§ (Sony §/WRTGATE§ functional counterpart) qualifies the write interval.

**[SONY]** 699-0452-A **sheet 29**, §3.4.5.1 defines:

| Event | Timing |
|---|---:|
| /READY valid before /WRTGATE assertion (T1) | **≥0.5 µs** |
| Internal drive /ERASE onset after gate (T2) | **0–43 µs** |
| Gate assertion → first WRDATA transition (T3) | **≥1.8 µs** |
| Last recorded WR transition → gate release (T4) | **≥2 µs** |
| Gate release → /READY deassertion (T5) | **≥0.5 µs** |
| Internal /ERASE trailing timing (T7) | **480–590 µs** |

**[DECISION]** **/ERASE is internal to the original physical drive, not a pin on the Macintosh Plus 20-pin connector.** Do not create an /ERASE connector signal or simulate its analog erase head. Gate-to-WR timing still matters for qualifying and bounding input capture; the erase-head-specific T2/T7 values are **historical reference**, not MCU flash erase delays.

**[SONY]** 699-0452-A **sheet 30**, §3.4.5.2 gives WRTDATA rising/falling-edge slope threshold-crossing variation **≤50 ns** and **2/4/6 µs nominal** transition separations. The STM32 must capture both polarities with adequate timing resolution after its chosen level-shifter/Schmitt filtering; it cannot infer a transition from the line's final sampled level alone.

**[DECISION]** WR/gate capture and RD generation share a coherent **virtual track phase**. A write gate need not begin exactly at a GCR codeword boundary; the firmware reconstructs full codewords from transitions, validates the stored sector header/received data field and publishes a coherent new standard track for immediate readback. The 50+ SYNC6 format-candidate rule and 703 GCR data/checksum codewords remain as established in Part 4 §12.

**[DECISION]** After a valid gate release, the host-visible **RD stream must resume within the specified response window**, including the original ROM's immediate post-format sector-0 address check. Flash persistence is **asynchronous**, and cannot be placed on the critical gate-release-to-RD path.

**[OPEN]** Measure the exact IWM /WRREQ pin behaviour, initial pipeline bit phase, gate release, WR edge jitter and read/write switching on the original host. The source specifies drive-side functional /WRTGATE; final assertion/deassertion translation must be established for the Plus's /WRREQ.

### 15.7 Command selector and LSTRB timing

**[SONY]** 699-0452-A **sheet 25**, §3.4.2, specifies the common command timing:

| Parameter | Source limit | Meaning |
|---|---:|---|
| T6 | **≥0.5 µs** | /ENBL active before selector/address |
| T1 | **≥0.5 µs** | Selector/address setup before LSTRB rising |
| T2 | **≥1.0 µs** | LSTRB high pulse width |
| T4 | **≤1.0 µs** | Selected control latch responds to rising LSTRB |
| T5 | **≤0.5 µs** | Latch responds to interface disable |

**[ROM]** The Plus uses two IWM PH3 accesses with two NOPs between them to generate LSTRB; command **changes are triggered on the rising edge**. See [Sony_Pulse](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1213-L1227).

**[DECISION]** Treat CA0/CA1/CA2/SEL as level-address/value inputs; sample only on **a qualified rising LSTRB** for a write to the selected drive latch. Repeated selector changes with LSTRB low are **status-source selection**, not a command. Register readback/status selection and command writes may share addressing; do not conflate them.

**[SOURCE DISCREPANCY]** Part 1 §6.3 records that the **1985 Apple 699-0321** command diagram prints **T4 ≥1.0 µs / T5 ≥0.5 µs**, while the **1986 699-0452-A** diagram prints **T4 ≤1.0 µs / T5 ≤0.5 µs**. We cite the **later 1986 target** as the electrical design response goal, but retain the discrepancy explicitly and require actual-interface validation. The 0.5-µs setup and 1.0-µs strobe-high minimum agree across both versions.

**[OPEN]** Choose hardware capture vs interrupt/peripheral path that cannot miss a qualifying strobe under simultaneous RD pulses, flash transfers or WR capture.

### 15.8 STEP, direction, TK0 and READY timing

**[SONY]** 699-0452-A **sheet 26**, §3.4.3.1 gives direction/step timing: /DIRTN stable **≥1.0 µs** before STEP (T1); the **/STEP low acknowledgment lasts 0.5–72 µs** (T2); other illustrated enable, direction and step separations are **≥0.5 µs**. /STEP is an **internal latched command/status** selected onto RD, **not** a separate connector pin.

**[SONY]** 699-0452-A **sheet 27**, §§3.4.3.2–3.4.3.3, gives:

| Reference characteristic | Value | Emulator interpretation |
|---|---:|---|
| /WRTGATE inactive before STEP | **≥620 µs** | Do not accept cross-track writes during a gate or violate write/step ordering |
| STEP → /READY not-ready transition | **≤150 µs** | Logical track readiness must not remain stale after accepted step |
| Settling after a normal one-track step | **≤36 ms** | Original **maximum**, not a required 36-ms emulator delay |
| Settling after a zone change | **≤152 ms** | Original **maximum**, not a simulated speed-change delay |
| Maximum-rate repeated steps | **≤600 ms** | Original **mechanism ceiling**, not a fixed digital waiting period |
| /TK0 assertion/deassertion near track-zero crossing (sheet 31) | **≤6 ms** | May update digital track status sooner; must remain consistent |

**[DECISION]** Apply **one logical radial step per qualified strobe**, clamp physical coordinate 0..79, provide **/STEP feedback** with compliant observable timing, and report /READY only when the newly selected cached track can be correctly served. An immediate logical track update is permitted; artificially waiting for a physical carriage is not.

**[DECISION]** A STEP, side change, or enable change must not corrupt an in-progress flash transaction or cause the current cached RD to expose a different track under the old phase/state. Part 1 owns exact command state, Part 6 owns unusual/conflicting control sequences.

**[SONY]** 699-0452-A **sheet 28**, §3.4.4, reports motor/insert /READY reference timing: motor-on to ready **≤600 ms**, removal /CSTIN transition to not-ready **≤0.5 µs**, new insertion to ready **≤1 s**, and a **≤50-ms** after-disable timing label (requires diagram-specific interpretation). Only **necessary valid host-visible state changes** are required from our emulator; maximum mechanical settling times are not minimum waits.

### 15.9 TACH timing and motor-status behavior

**[SONY]** TACH status corresponds to **60 pulses per nominal physical revolution**. The five standard logical zones use approximately **394 / 429 / 472 / 525 / 590 pulses/s** (same numerical frequencies as reference RPM), selected by the track descriptor. The Sony TACH description cites **±0.2% period accuracy**; different ±2.5%/±1.5% mechanical speed limits must not be mistaken for the pulse generator's tolerance.

| Tracks | Sectors/side | TACH pulse frequency | Nominal period |
|---|---:|---:|---:|
| 0–15 | 12 | 394 Hz | 2538.07 µs |
| 16–31 | 11 | 429 Hz | 2331.00 µs |
| 32–47 | 10 | 472 Hz | 2118.64 µs |
| 48–63 | 9 | 525 Hz | 1904.76 µs |
| 64–79 | 8 | 590 Hz | 1694.92 µs |

**[ROM]** The Plus ROM **does measure TACH on 800K read paths**, despite bypassing old 400K PWM speed adjustment. [Sony_ReSeek and speed measurement](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L832-L882), [Sony_FigTrkSpd / Sony_WaitTach](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1879-L1988).

**[DECISION]** TACH is **a selected periodic status level on RD**, not a separate wire and not a permanently fixed 0/1. The host samples its current waveform; **selecting TACH must not restart the phase**. Do not derive five independent GCR cell clocks from TACH. One globally calibrated nominal 489,600-bit/s data clock remains in effect for every track.

**[PROPOSED]** An independent timer generates the five TACH waveforms and the RD mux presents the current instantaneous value. A provisional **~50% duty** permits the Plus ROM's repeated low/high samples, but its precise duty and idle level with motor stopped remain **[OPEN]** and must be checked experimentally.

### 15.10 Media presence, eject and power-fail timing

**[DECISION]** The selected RD status functions /CSTIN, /WRTPRT and /READY must be mutually consistent with **qualified cartridge presence**, active write permission, correct track buffer and actual streaming readiness (Part 1 §9 and Part 6 §14). A healthy unformatted cartridge is present but cannot supply counterfeit valid GCR sectors. Host logical eject and physical cartridge withdrawal are distinct.

**[SONY]** 699-0452-A **sheet 32**, §3.4.7 gives original EJECT/MOTORON reference: motor-off before eject **≥200 ms**, LSTRB high **≥1.0 µs** (diagram also shows **300-ms maximum**), /CSTIN deassertion after eject **≤1.5 s**, and 150-µs minimum enable relationship. **Sheet 35**, §3.4.10, shows an original mechanical **EJECT** activation interval **490–510 ms** and /CSTOUT response **≤1.5 s**.

**[DECISION]** These describe the **original eject mechanism**, not a requirement to synthesize a physical solenoid's 500-ms activity. The emulator must instead obey **software-eject semantics and host timing compatibility**, preventing a **successful logical eject before accepted dirty writes are durable or recoverable**. There is no host-level generic “flash busy” status, so the PCB/firmware must achieve a valid safe-release procedure within the applicable ROM-visible window; simply stretching an unrelated pin or inventing a status does not solve it.

**[DECISION]** Unexpected cartridge disconnection must invalidate old RD/status outputs, stop new SPI commands to unstable contacts, prevent delayed stale-cache writes to a replacement cartridge and enter a safe state. **[OPEN]** Cartridge sensor timing, power hold-up and exact status/error sequencing depend on Section 16 and the mechanical contact design.

### 15.11 Power rails, protection and mixed-voltage architecture

**[APPLE]** Macintosh Plus internal connector provides **+5 V (pin 11), +12 V (pins 13, 15, 17, 19)** and **−12 V (pin 9, intentionally unconnected)**. All grounds 1/3/5/7 are return conductors. The connected-machine *aggregate* load budget in the Apple Guide is **600 mA at +12 V, 700 mA at +5 V, 10 mA at −12 V**; original 800K drive reference load **600 mA at +12 V and 360 mA at +5 V**. These are **historical source limits**, not a target for emulator power consumption.

**[DECISION]** Circuit design must remain within **the total host load budget**, including any simultaneously connected external peripherals, and ideally use materially less power than the original drive. The chosen rail(s), regulator(s), STM32 core supply, SPI-cartridge supply and connector protection are **not yet selected**. **Do not tie +12 V to +5 V or use −12 V/pin 9.** Unused +12-V pins are not arbitrary signal pins; terminate them only as the final power design specifies.

**[DECISION]** No 5 V line may be connected to a non-5-V-tolerant STM32 pad. Verify compatibility of every input/output **for powered, resetting, unpowered and hot-removal states**. Prevent positive/negative back-power through ESD diodes, level translators or SPI cartridge contacts. RD output must default to high impedance on MCU reset and when the host drive is disabled.

**[PROPOSED]** Use input buffer/translator(s), a dedicated fast tri-state RD driver, clean regulator with power-good/brownout supervision, adequate local bypass/decoupling, appropriate overcurrent/ESD protection and connector/contact qualification. Optional hardware RD-disable gating should respond even with stalled firmware. Choose component thresholds, impedance, edge rates and RC filtering only after worst-case RD pulse and selector propagation analysis—**large RC filters can destroy 0.3–0.8-µs pulses and 0.5-µs response limits**.

**[OPEN]** Select supply strategy, 5 V driver/translators, brownout detection, contact/ground sequencing, clamp/ESD behavior, current draw and whether cartridge removal needs hold-up energy before design approval.

### 15.12 Interface implementation boundaries and timing priority

**[DECISION]** Maintain strict timing isolation:

~~~text
Mac Plus PH0/1/2/SEL/PH3 ----> 5-V-safe fast receiver ----> selection/strobe logic
Mac Plus /ENBL -------------> fast output-enable safety path ----+
Mac Plus /WRREQ + WR -------> qualified both-edge timestamp capture / DMA
                                                                  |
STM32 standard track buffers (>=3 × 12 KiB) -> GCR timer/serializer ----> RD source mux
TACH timer/selected status levels -----------------------------------> RD source mux
RD source mux + host enable -----------------------------------------> 5-V-safe tri-state driver -> Pin 16 RD

SPI flash on removable cartridge <-> independent background storage worker
        (must not stall output timing, command sampling, input captures)
~~~

**[DECISION]** Highest-priority hardware paths are **RD output disable**, **selector-to-RD response**, **RD pulse timing**, **LSTRB qualified event capture** and **WR both-edge timestamp capture**. TACH remains deterministic. SPI erase/program, directory commits, media import and diagnostics are not allowed to perturb these paths.

**[DECISION]** Keep **at least three complete >=12-KiB track buffers**, separate capture DMA/rings and asynchronous flash worker, as previously agreed. Present a reconstructed standard ROM-compatible track; no raw nonstandard/copy-protection playback is required in the first release (Part 6 §14).

**[OPEN]** Specific STM32 part, available timers/input capture/DMA, glue-logic size, clock synchronization, buffer OE and RD serializer circuit are the purpose of Section 16. This section intentionally does **not** approve an MCU part number, pin assignment, schematic or PCB layout.

### 15.13 Bench verification and acceptance matrix

**[DECISION]** Do not connect an unqualified prototype to a working Macintosh Plus. First verify the board on a safe, current-limited bench setup with a host-waveform fixture, then with the actual Plus and logic analyzer/scope.

| Verification | Pass criterion / evidence |
|---|---|
| Harness / pin orientation | Confirm key/pin 1 against original board; +5/+12/−12/GND correctly mapped; pin **9/20 physically disconnected** |
| Host input logic | Meet original Sony **VIH ≥2.2 V / VIL ≤0.8 V** thresholds and accepted input loading under temperature/supply conditions |
| RD electrical drive | **VOH ≥2.4 V at −1 mA; VOL ≤0.5 V at 8 mA** for host-facing load |
| RD disabled/high-Z | RD **high impedance within ≤0.5 µs** of /ENBL inactive, including MCU reset/firmware stall |
| Status source switch | Host CA/SEL change → correct RD status **≤0.5 µs**, without spurious LSTRB command |
| RD enable | /ENBL active → selected RD valid **≤0.5 µs** |
| Serial RD pulses | Width **0.3–0.8 µs**, correct GCR transitions and source/pulse spacing at calibrated bit clock |
| Serial source selection | Head/data switch and /READY/data timing per sheet 24; no stale track |
| RD after write | **≤620 µs** for valid data resumption, including post-format sector-0 readback |
| LSTRB qualification | ≥0.5 µs setup, ≥1.0 µs pulse, ≤1.0 µs target latch response (1986 target) |
| STEP feedback | Correct one-step-per-strobe, observable **0.5–72 µs** /STEP low pulse and logical TK0/READY |
| WR capture | Record **both edges** at all legal transition intervals; no DMA overrun, no false sector write |
| Write gate | Respect real gate/WR phase, minimum setup/trailing conditions, no invented /ERASE wire |
| TACH | 394/429/472/525/590-Hz target, accurate pulse periods and phase independent of status selection |
| Motor/READY/presence | Proper ROM-visible READY, no mechanical wait; fast invalidation on media removal |
| Cartridge power/removal | No MCU/host back-power, false RD drive, unsafe flash command or stale writes across removal |
| Flash stress interference | RD jitter, selector response, WR capture and TACH remain compliant throughout SPI erase/program |
| ROM interaction | Real Plus boot, read, sector write, 400K/800K format, immediate format readback, eject and fault paths |

**Instrumentation:** high-impedance active/passive probes with known loading, sufficiently fast oscilloscope for 50-ns edges/sub-µs RD pulse widths, logic analyzer for long serial/selector traces, repeatable test pattern generator and current-limited bench supply. Record **at the host connector**, not solely MCU timer registers, and include propagation delays through buffers.

**[OPEN]** Select numerical **total RD edge jitter**, allowable clock-drift/temperature budget, controlled probe loading and the exact trigger/alignment criteria from measured IWM behavior. Do not invent a jitter tolerance not specified by the source.

### 15.14 Decisions, discrepancies and open implementation work

**Settled [DECISION]:**

- Target **Macintosh Plus internal 20-pin MFD-51W** host interface; **pins 9 (−12 V) and 20 (PWM) disconnected**; separate external DB-19 later.
- All eight host-facing signal inputs use safe electrical levels, and **pin 16 RD** is the only drive output; RD has correct multiplexing and **true high-Z when /ENBL is inactive**.
- Use the original **DC level/current guarantees** and **sub-microsecond selector/status/enable limits**; RD 0.3–0.8-µs timed GCR pulses and proper WR **both-edge** capture.
- Original 1986 LSTRB setup/high-width/response requirements apply, with the 1985/1986 diagram discrepancy recorded.
- Logical STEP/TK0/READY, TACH waveform and cartridge/eject status are valid Sony host behaviour, with **no invented mechanical waits**.
- One global 489,600-bit/s nominal GCR cell rate, five correct TACH profiles and standard structured-GCR-only first release.
- Background SPI flash operations may **never interrupt the real-time electrical contract**.

**Open implementation/validation (not new format decisions):**

1. Exact MCU, timers/DMA/mux/output-enable architecture and buffer-chip electrical specifications.
2. Plus host actual RD load, pull-ups, translator thresholds, all rail power budgets and unplugged/reset back-power safeguards.
3. Measured host clock/GCR variation, RD polarity/edge jitter and both-edge WR gate/capture phase.
4. Confirmation/handling of 1985 vs 1986 T4/T5 command diagram discrepancy.
5. ROM and host behavior with no motor, stopped TACH duty/idle, status-mode transitions, not-ready after step and blank media.
6. Power-fail/dirty-cache-safe eject within host-compatible timing; connector physical orientation and sensor/release strategy.

**Document history**

- **2026-11-11:** Created Part 7 / §15 from Apple Macintosh Plus Table 9-3 and Apple/Sony 699-0452-A sheets 21–32/35, reconciled with existing Part 1 and Part 2 measurements/specification tables. Locked 5 V-era electrical levels and high-Z/fast RD requirements, 489,600-bit/s serial timing, WR edge/gate characteristics, strobe/step/READY/TACH/eject constraints, and clean Section 16 implementation boundary.
