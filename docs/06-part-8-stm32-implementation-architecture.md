# Sony MFD-51W Drive Interface Protocol — Part 8: STM32 Implementation Architecture

**Document:** 06 — Part 8  
**Protocol section:** **16 — STM32 implementation architecture**  
**Status:** Proposed implementation architecture grounded in approved requirements; MCU, pinmux, circuit, read engine and flash-commit algorithm **not yet approved**  
**Target:** Macintosh Plus M0001AP internal 20-pin Sony MFD-51W emulator  
**Date:** 12 November 2026

> **Sequence:** [Document 02](02-macintosh-20-pin-interface.md) owns the internal connector; [Part 1 / Sections 1–9](06-part-1-drive-interface.md) drive control/status; [Part 2 / Section 10](06-part-2-serial-gcr-read.md) GCR read stream; [Part 3 / Section 11](06-part-3-media-architecture.md) sector records and cartridge; [Part 4 / Section 12](06-part-4-serial-gcr-write.md) WR capture and persistence; [Part 5 / Section 13](06-part-5-format-geometry-400k.md) standard reconstruction; [Part 6 / Section 14](06-part-6-error-handling-standard-media.md) faults; [Part 7 / Section 15](06-part-7-electrical-timing.md) electrical timing. **This Section 16 maps those fixed contracts to candidate hardware and firmware.** Future Section 17 validates them.

**Labels:** **[DECISION]** settled project requirement; **[SOURCE]** published specification; **[CANDIDATE]** implementation proposal to investigate; **[OPEN]** unresolved selection or measurement. A candidate is not a finalized BOM, pinmux or PCB.

## Contents

- [16.1 Fixed constraints and scope](#161-fixed-constraints-and-scope)
- [16.2 Proposed hardware blocks](#162-proposed-hardware-blocks)
- [16.3 Candidate STM32 and peripheral selection gate](#163-candidate-stm32-and-peripheral-selection-gate)
- [16.4 Fast RD selector and electrical isolation](#164-fast-rd-selector-and-electrical-isolation)
- [16.5 Serial GCR RD generation](#165-serial-gcr-rd-generation)
- [16.6 WR both-edge capture and decoding](#166-wr-both-edge-capture-and-decoding)
- [16.7 Shared virtual state and track publication](#167-shared-virtual-state-and-track-publication)
- [16.8 LSTRB, STEP, READY and TACH](#168-lstrb-step-ready-and-tach)
- [16.9 RAM and DMA budget](#169-ram-and-dma-budget)
- [16.10 SPI-flash access and concurrent reads](#1610-spi-flash-access-and-concurrent-reads)
- [16.11 Durable writes and recoverable metadata](#1611-durable-writes-and-recoverable-metadata)
- [16.12 Cartridge contacts, power and eject](#1612-cartridge-contacts-power-and-eject)
- [16.13 Firmware ownership and priorities](#1613-firmware-ownership-and-priorities)
- [16.14 Development stages and acceptance gates](#1614-development-stages-and-acceptance-gates)
- [16.15 Approved boundaries and pending choices](#1615-approved-boundaries-and-pending-choices)

---

## 16. STM32 implementation architecture

### 16.1 Fixed constraints and scope

**[DECISION]** The emulator replaces the **Sony drive mechanism**, not the Macintosh floppy controller. The original IWM, ROM, CPU and drive commands remain in use. It supports **standard 400K/800K Macintosh disks**. The RAW_TRACK representation ID is reserved, but nonstandard/copy-protected track playback and bit-level RAW_TRACK writing are **not part of the first firmware**.

| Requirement | Fixed contract |
|---|---|
| Macintosh interface | Internal 20-pin; **pin 9 −12 V disconnected**, **pin 20 PWM disconnected** |
| Host inputs | PH0, PH1, PH2, PH3/LSTRB, SEL, /ENBL, /WRREQ, WR |
| Host output | **RD pin 16 only**; status/data multiplexed; true high-Z when disabled |
| RD control | Normal enable/disable and status selection **≤0.5 µs** per Section 15 |
| GCR | **489,600 bit cells/s nominal**, 2.042484 µs/cell, one global calibration |
| RD pulses | **0.3–0.8 µs** width, correctly timed for encoded 1 bits |
| WR | Capture **both level-transition polarities** under qualified gate |
| Storage | **32 metadata bytes + 703 GCR codewords = 735 bytes/sector**, complete encoded headers retained |
| Track | Full standard cyclic bitstream in RAM, logical buffer starts at sector 0, RD phase independent |
| SRAM | **At least three complete ≥12-KiB track buffers**, plus capture/staging |
| Cartridge | **16 Mbit / 2 MiB** removable SPI NOR, erase-aligned standard track extents |
| TACH | Selected periodic status **394/429/472/525/590 Hz**, independent of data-clock division |
| Errors | No silently lost accepted writes; protected sectors/tracks, real safe-eject/recovery design |

**[DECISION]** Simulated motor motion, artificial seek delay, rotational latency and index pulses are unnecessary. Electrical minimum response times and serial pulse timing are mandatory. SPI erasing, flash polling, formatting checks and diagnostics **must not perturb the real-time IWM interface**.

### 16.2 Proposed hardware blocks

~~~text
                MACINTOSH PLUS ORIGINAL IWM
                INTERNAL 20-PIN DRIVE CONNECTOR
  PH0 PH1 PH2 SEL PH3  /ENBL  /WRREQ WR       RD
          |                |       |          ^
          v                v       v          |
    5-V-safe input    fast enable    both-edge timer
          |           protection     + capture DMA
          v                |             |
    fast selector     RD output-OE        v
    (status/data)          |         timestamp ring
          ^                |             |
          |                |             v
      status latches       |       WR field parser
          ^                |       and validator
          |                |             |
       STM32 state <-------+-------------+
          ^                              
          |             >=3 FULL CYCLIC TRACK SRAM BUFFERS
   TACH timer and          ^                |
   command timers          |                v
          |          track constructor   GCR serializer
          |          with 735-byte       + pulse shaper
          |          sector records           |
          +-------> fast RD status/data mux <-+
                            |
                    tri-state 5-V RD driver
                            |
                        PIN 16 RD

          priority-aware SPI NOR worker / track directory
                          |
          removable 3.3-V 2-MiB SPI flash cartridge

RD enable/status selection and pulse/WR timing must not
wait for foreground firmware or flash erase/program.
~~~

**[CANDIDATE]** Use one STM32 to coordinate drive state, sector reconstruction, DMA and storage, with **a small external fast-mux/level-buffer stage** where needed for genuine ≤0.5-µs RD selection and fail-safe high-Z. A CPLD/FPGA is **not** a current requirement. If the MCU timer/serial-shifter experiment fails timing, evaluate narrowly scoped glue logic rather than silently relaxing the specification.

### 16.3 Candidate STM32 and peripheral selection gate

**[CANDIDATE]** Investigate the **STM32H723ZG** first. ST's published product information lists **Cortex-M7 up to 550 MHz**, **1 MiB embedded flash**, **564 KiB total SRAM**, numerous timer/capture channels, DMA controllers and **two Octo-SPI interfaces**. This combination appears ample for the proposed full-track cache and independent signal engines, but no MCU has yet been selected.

Sources: [ST STM32H723ZG product](https://www.st.com/en/microcontrollers-microprocessors/stm32h723zg.html), [STM32H723 family datasheet](https://www.st.com/resource/en/datasheet/stm32h723ze.pdf).

**[OPEN — MCU approval prerequisite]** Verify the *exact package* and every peripheral combination with ST's reference manual and CubeMX/pinmux, not merely a feature list:

- Input path for four selector bits and LSTRB, ensuring fast status multiplexing and correctly qualified commands.
- A real-time RD serializer with pulse shaping and true host enable gating.
- Timer channels capturing **both WR edges**, preferably tied to a 32-bit free-running timestamp counter, plus gate timing.
- Independent timer for 394–590-Hz TACH and bounded STEP feedback.
- DMA channels that can operate simultaneously with RD, WR, SPI and eventual debugging.
- A DMA-accessible SRAM map, cache-coherency plan and clock tree able to produce one accurate **489,600-bit/s** nominal cell rate.
- Complete 5-V input-output electrical interface, cartridge SPI pin group, SWD and reset/boot/power supervisors.
- Host and cartridge rail consumption under worst-case operating and power-fail conditions.

**[DECISION]** MCU selection is **not determined by core MHz or nominal SRAM capacity alone**. A smaller STM32 or an alternate H7 package can be compared, but the first candidate must pass the timing/pinmux matrix before committing a BOM or KiCad layout.

### 16.4 Fast RD selector and electrical isolation

**[DECISION]** Host PH0–PH3, SEL, /ENBL, /WRREQ and WR are **5-V-era inputs**; choose genuine compatible receivers/translators. Only RD is driven to the host. The RD driver must meet the voltage/current limits in Section 15, multiplex status and pulse sources, and become **high impedance within 0.5 µs** after /ENBL goes inactive, including during reset or a stalled MCU.

**[CANDIDATE]** Store the current drive/status bits in low-latency latches or stable hardware-visible state; apply a fast 16-selection mux to those bits and the live GCR signal. Host selector changes must respond in the Section 15 window without depending on a slow MCU GPIO polling loop. Have an asynchronous gate from /ENBL directly to RD output enable. Selecting a status must never produce a STEP/EJECT command: **LSTRB rising edge is the command event**.

**[OPEN]** The chosen logic ICs, thresholds, propagation delay, output drive, high-Z turn-off, line loading and power-off behaviour require a complete **host-connector-to-RD measurement**, not merely a timer register simulation. Validate the physical host's input levels with high-impedance instrumentation.

### 16.5 Serial GCR RD generation

**[DECISION]** The read engine outputs a fully reconstructed cyclic standard GCR stream from a **≥12-KiB RAM buffer**. A recorded **1** bit produces an **RD pulse**; a 0 does not. Each bit cell lasts nominally **2.042484 µs**, but a pulse must be only **0.3–0.8 µs** wide. The read playback phase advances independently of the logical buffer origin at sector 0.

Two **candidate alternatives**, neither yet approved:

**A — DMA-driven synchronous bit shifter and narrow hardware strobe.** A SPI/USART-synchronous shifter or comparable serial peripheral emits one bit per calibrated cell, while a synchronized **narrow strobe** gates the bit value into a real RD pulse, even for consecutive 1 bits. A raw SPI MOSI waveform holding high for a whole bit cell, or its standard half-cycle clock pulse, is **not** automatically a compliant 0.3–0.8-µs RD output. Prove setup/hold and pulse shape, DMA refill, wrap phase, and **non-byte-aligned final circular bit length** handling. An 8-bit serial shifter transfers approximately **61,200 bytes/s** at nominal clock.

**B — Timer-output-compare with DMA pulse-event queue.** Scan the prepared cyclic bitstream into a bounded ahead-of-time pulse-position queue. Timer compare and hardware pulse-width logic emit pulses without a CPU ISR for each pulse. The worst-case event rate approaches **489,600 events/s**; test DMA refill, bus contention, timer register update semantics and seamless wrap.

**[DECISION]** The chosen method must satisfy pulse width, position, exact SYNC6/gap content, absence of missing/extra transitions, status-source selection and uninterrupted timing during SPI flash operations. Use **one global calibrated GCR cell clock**, never five per-zone data clocks. Do not assume hardware SPI can implement arbitrary bit-count wrapping without additional logic.

### 16.6 WR both-edge capture and decoding

**[DECISION]** Macintosh WR is **level-transition encoded**; **both rising and falling** transitions represent data. Qualify the capture by /WRREQ, host enable, selected cartridge/side and protection. WR capture must not depend on SPI operations or per-edge CPU interrupts.

**[CANDIDATE]** A free-running **32-bit input-capture timer plus DMA ring** is the primary experiment. Check whether an appropriate timer channel truly supports both-edge capture, or whether opposite-polarity channels can be timestamped against one clock and merged without ordering ambiguity. Use the same clock domain, or a proven synchronized correlation, for gate-on/gate-off timestamps and RD phase.

**[REFERENCE]** Worst-case one transition per bit cell means **489,600 transitions/s** and about **1.96 MB/s** of 32-bit timestamps. A **4,096-entry timestamp ring lasts ~8.37 ms** at that rate, so DMA half-buffer handling and continuous decoding need measured bounded latency. Two 4,096-entry rings occupy **32 KiB**. A track format burst lasts roughly 120–171 ms; it cannot fit in a small finite edge buffer without continuous processing.

Capture flow:

~~~text
qualified /WRREQ falling edge -> latch cartridge/track/side/RD phase
    -> capture both WR edges and classify GCR sync
    -> one SYNC6 + D5 AA AD: ordinary sector candidate
    -> >=50 consecutive SYNC6: whole-track format candidate
    -> recover byte alignment; validate stored address/sector context
    -> validate complete data header + 703 GCR bytes/checksums
    -> update/rebuild coherent RAM track (not arbitrary bit splice)
    -> queue background safe persistence
~~~

**[DECISION]** Never assume gate edges align with codeword boundaries. Preserve original complete address headers on ordinary sector writes, and do not accept incomplete/invalid sectors. ROM format retry replaces the candidate track and must be readable immediately after capture, before flash commit. Real host gate timing and edge polarity require measurement.

### 16.7 Shared virtual state and track publication

**[DECISION]** Maintain these logically distinct records:

- **Media:** cartridge identity/generation, qualified presence, 400K/800K capacity, protection, blank/valid/fault and eject status.
- **Drive:** PH selector, enable, motor and direction latches, logical radial position **0–79**, side, STEP pulse, TK0/READY, TACH phase.
- **Track:** STRUCTURED_GCR descriptor, sector count/order, GapSync, valid circular bit count, **physical-sector-ID to bit-position map**, RAM buffer ID and RD phase.
- **Write:** latched target cartridge/track/side/phase; DMA capture generation, incoming data/format candidate and dirty track version.
- **Flash:** track allocation, committed/pending generations, directory transaction, erase/program/verify state and safe eject eligibility.

**[CANDIDATE]** Publish a small **atomic track-generation descriptor** only when the selected buffer is complete and internally consistent. Pin active RD buffers and the only dirty generation; never mutate a buffer currently being serialized. Read-after-write sees the latest **accepted RAM generation**, not the previous NOR copy. A status/side selection does not reset an unrelated free-running TACH phase.

**[OPEN]** Prove phase continuity and ready behavior during real Plus side changes and cache misses; if a requested track cannot be served, do not output stale data under the new track identity.

### 16.8 LSTRB, STEP, READY and TACH

**[DECISION]** Sample commands on **qualified LSTRB rising edges** with the source-defined selector setup and ≥1.0-µs high pulse. Do not infer a command from selecting a status. STEP advances exactly one logical track according to direction and exposes an observable **0.5–72-µs internal /STEP feedback** through RD status; the MCU need not simulate a moving carriage.

**[DECISION]** READY is a genuine transfer-permission status: it can assert only when cartridge, motor command, selected track/side and output bitstream are coherent. A track change must not expose an older cached track. A selected 400K image on an 800K-capable drive does not magically contain recorded side-1 data.

**[DECISION]** Generate TACH with an **independent periodic timer** using 394/429/472/525/590-Hz reference outputs, **not** from five GCR data clocks. The ROM actually checks TACH during 800K reads. Selecting TACH must not restart its phase; the proposed ~50% duty cycle and stopped-idle level require physical comparison before approval.

**[DECISION]** /ENBL inactive requires fast RD high-Z and the correct motor-off latch effect. It does not authorize discarding an accepted dirty generation or aborting a safe flash transaction.

### 16.9 RAM and DMA budget

**[DECISION]** Provide **three complete independently allocated track buffers ≥12 KiB each (36 KiB)** plus separate time-critical capture and flash staging. Roles are dynamic: active read, incoming reconstruction, pending persistence; a dirty or currently emitted generation cannot be evicted.

**[CANDIDATE — illustrative starting allocation, not a frozen memory map]**

| Consumer | Suggested amount |
|---|---:|
| Three cyclic full-track buffers | 36 KiB |
| Two × 4,096-entry × 32-bit WR timestamp rings | 32 KiB |
| GCR record assembly / parser workspace | 16 KiB |
| GCR serializer/event staging | 16 KiB |
| SPI flash pages / read-modify-erase workspace | 16 KiB |
| Compact directory, sector and cache indices | 16 KiB |
| Stacks, diagnostics, task and spare space | 64 KiB |
| **Illustrative aggregate** | **196 KiB** |

**[SOURCE / CRITICAL]** A Cortex-M7 STM32H7's **TCM vs AXI/AHB SRAM domains and D-cache coherency** matter: *total 564 KiB RAM* is not necessarily accessible by all DMA controllers. Place DMA rings and serializer feeds in **DMA-reachable memory**, verify bus connectivity, and use cache maintenance or MPU-marked non-cacheable aligned regions; DTCM is often CPU-only for general-purpose DMA. ST's own DMA guidance notes such access/cache hazards. [ST H7 DMA-memory discussion](https://community.st.com/stm32-mcus-embedded-software-32/h743zi2-sdmmc-with-dma-mdma-28079).

**[OPEN]** Exact linker sections, cache lines, DMA controller reachability, concurrent bus bandwidth and stack peaks must be demonstrated for the selected STM32 package.

### 16.10 SPI-flash access and concurrent reads

**[DECISION]** The cartridge uses **16-Mbit (2-MiB) SPI NOR** and standardized **735-byte records** in 4-KiB erase-aligned track allocations. **SST26VF016B** remains a candidate chip, not the committed flash BOM.

**[SOURCE]** [Microchip SST26VF016B datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/20005262D.pdf) specifies **4-KiB erases ≤25 ms**, **256-byte page programs ≤1.5 ms**, and **Write-Suspend / Write-Resume**. A suspended erase can permit reading *different, unaffected sectors*, not the erase target. There is **only one suspended write at a time**, a **minimum 500 µs** interval between Write-Suspend commands, and **up to 25 µs** suspend latency.

| Sector count | Record bytes | Page programs | 4-KiB erase units | Reference internal max |
|---:|---:|---:|---:|---:|
| 12 | 8,820 | 35 | 3 | 127.5 ms |
| 11 | 8,085 | 32 | 2 | 98.0 ms |
| 10 | 7,350 | 29 | 2 | 93.5 ms |
| 9 | 6,615 | 26 | 2 | 89.0 ms |
| 8 | 5,880 | 23 | 2 | 84.5 ms |

These times exclude SPI transfer, poll overhead, metadata transactions, verification and retry.

**[CRITICAL ARCHITECTURAL RISK]** A single NOR chip is **not freely readable while busy erasing/programming**. SRAM protects active RD, but a **new track request on a cache miss** may collide with flash work. Write-suspend helps but its latency/restrictions and only three buffers mean seamless cache-miss handling is **not yet proven**.

**[CANDIDATE]** One priority-aware flash worker serializes all commands. An urgent read of unaffected data may suspend a safe ongoing program/erase, wait for the chip's actual ready state, fetch records and resume. Prefetch likely alternate head/neighbor tracks when spare RAM is available; **never assume prefetch succeeds** or read a suspended erase target. Reconcile host READY timing and track-selection changes with real Macintosh tests.

### 16.11 Durable writes and recoverable metadata

**[DECISION]** **Accepted** means a complete validated WR field published coherently in RAM; **durable** means its flash content and recovery metadata have been safely verified. RAM is not durable under arbitrary power loss. An accepted dirty track cannot be evicted as clean and must not silently disappear at a successful software eject.

**[DECISION]** Preserve centralized versioned directory, one global GCR timing record, independent erase-aligned extents and per-track/cartridge generations. The full 800K track allocations occupy **1,408 KiB** of the 2,048-KiB chip, leaving **640 KiB before metadata, spare extents and journal**; this capacity margin is not proof of crash-safe persistence.

**[CANDIDATE]** Evaluate **copy-on-write (COW) complete-track replacement plus a compact append-only descriptor journal/checkpoint**. For a sector update:

~~~text
validate incoming ordinary sector
 -> update/pin latest RAM track generation for immediate readback
 -> allocate a spare isolated erase-aligned track extent
 -> erase + program + verify new structured sector records
 -> append / verify small durable generation+extent commit record
 -> only then make flash directory reference the new generation
 -> reclaim previous extent when no recovery reader needs it
~~~

A full ROM format may use a specially isolated destructive target policy, but cannot damage other track extents or directory. A journal avoids rewriting an entire large directory for each sector write. **[OPEN]** Prove spare allocation, atomic commit marker, wear, flash page/sector constraints, directory checkpointing, repeated write throughput and recovery on power loss before approving a final storage ABI.

**[IMPORTANT LIMIT]** The Sony interface has no host-facing generic **write ACK or pause**. Do not promise that every accepted SRAM write survives instantaneous unannounced power removal unless the supply/hold-up design proves that guarantee. Safe software eject and recoverable committed generations remain mandatory requirements.

### 16.12 Cartridge contacts, power and eject

**[DECISION]** Physical removal and logical software eject are different events. Qualify cartridge identity and contacts before enabling SPI or reporting mounted media; blank healthy media is present and format-capable, while failed media is not silently treated as blank.

**[CANDIDATE]** Design staged ground/power contacts, presence/ID, controlled CS/SCK/IO, optional write-protect contact, a regulated **3.3-V cartridge rail** and appropriate hot-unplug protection. Every queued transaction must retain a cartridge-generation token, preventing a delayed old-media write to a replacement cartridge.

**[DECISION]** On eject: block new writes, complete/make accepted writes recoverable, then signal genuine no-media. Unexpected removal: stop using unqualified SPI pins, suppress stale RD and invalidate the old generation. A pending write or fast cartridge change may require separate contact sensor/interlock/hold-up hardware; actual component selection is open.

**[CANDIDATE]** Start the power budget with Macintosh **+5 V** into a suitable 3.3-V regulator; +12 V might remain unused but must stay a supply pin, never a GPIO. **Pin 9 −12 V and pin 20 PWM are unconnected.** Provide RD power-off high-Z, brownout/reset supervisor and no back-power through ESD diodes or cartridge contacts. Selection of regulator, protection, output buffer and hold-up energy is deferred until worst-case current/rom-eject window analysis.

**[NEW TEST RESOURCE]** A Macintosh **M0001AP with an included external drive** has been acquired and is expected soon. It will be useful for actual IWM/RD/WR/TACH measurements and comparison. **Inspect and identify both drives first**; an external DB-19 mechanism/revision must not be assumed to have the same internal 20-pin signal mapping, and advertised working condition is **not yet a verified bench result**.

### 16.13 Firmware ownership and priorities

**[CANDIDATE]** Begin with bare-metal/event-driven firmware or a minimal RTOS only when latency tests justify one. Critical host signal timings should be **peripheral/hardware driven**, not one CPU ISR per GCR pulse/WR edge.

| Priority | Owner | Cannot wait for |
|---|---|---|
| Fast hardware | RD high-Z /ENBL, selector-to-RD mux | CPU, interrupts, SPI |
| Hard timer/DMA | GCR pulse timing, WR timestamp capture, free-running TACH | Flash erase, logs, decoding |
| Short bounded event | LSTRB, STEP feedback, gate edges, track publish | Filesystem, directory walk |
| Bounded software | Drain WR rings, GCR validation, track build | Busy flash on an already-ready RD path |
| Background | SPI read/erase/program, COW/journal, prefetch, diagnostics | Serial timing and gate capture |
| Offline PC | Logical image import and 6-and-2 conversion | STM32 real-time read |

**Invariants:** single owner of the cartridge SPI command stream; DMA-visible buffers mapped/coherent; no mutation of active RD track; dirty generations pinned; every queued request tagged with cartridge identity; no false-ready stale track; no silently dropped write if all buffers are exhausted. Diagnostics must be buffered or disabled during timing-sensitive work.

### 16.14 Development stages and acceptance gates

**[DECISION — recommended sequence]**

| Stage | Implementation/test | Exit evidence |
|---|---|---|
| P0 — acquired Mac inspection | Verify M0001AP and external drive identity, connector orientation and safe rail measurements | Documented host harness/power; no risk to historic hardware |
| P1 — RD selector interface | Fast selector, true high-Z enable, status latches, LSTRB and STEP | ≤0.5-µs mux/OE, correct host DC drive and timing |
| P2 — GCR output | Test SPI+strobe and/or timer/DMA output with SYNC6 and full track | Correct pulse width, calibrated cell clock, cyclic wrap and no extra pulses |
| P3 — WR capture | Both-edge DMA ring, gate timing, sector ID and 50-SYNC6 format recognition | No missed edges, complete sector/format capture and immediate RAM readback |
| P4 — cache/flash concurrency | ≥3 track buffers; 2-MiB NOR; flash suspend/read collision | Valid ROM-like stream throughout flash work; no stale RD |
| P5 — recovery | Repeated writes, mid-erase power cut, protect, surprise removal, dirty eject | Recoverable directory, no unrelated-sector corruption and defined limits |
| P6 — real Plus verification | Boot, 400K/800K read/write/format/Verify, TACH recheck | Logic-analyzer trace and Macintosh-visible success |
| P7 — hardware freeze | Select exact MCU, pinmux, buffers, power and cartridge circuit | Reviewed schematic/BOM/pin map for KiCad |

**[OPEN]** Section 17 owns complete automated test vectors, MCU stress benchmarks, actual clock/jitter tolerance, flash recovery fault injections and realistic host timing acceptance. A headline 550-MHz CPU rating is never a substitute for the P1/P2/P3 timing gates.

### 16.15 Approved boundaries and pending choices

**Settled constraints retained:** original Macintosh IWM/ROM; internal MFD-51W 20-pin, **pins 9/20 left disconnected**; standard 400K/800K only; structured 32+703-byte sector records; exact reconstructed logical track; nominal single global GCR cell clock; independent five-profile TACH; **≥3 ×12 KiB** complete-track RAM buffers; independent SPI NOR persistence and genuine Sony electrical timing.

**To evaluate, not yet approve:** **STM32H723ZG**, fast 5-V-safe RD selector/buffer, hardware shifter+strobe vs compare/DMA pulse engine, both-edge input capture+DMA, exact 196-KiB illustrative SRAM placement, SST26VF016B suspend/resume, COW+journal flash transaction, 5-V-to-3.3-V power and safe cartridge contacts.

**Critical open gates:** exact MCU pinmux and DMA memory reachability; pulse/gate jitter with concurrent SPI; real host selector/read handoff; flash-busy cache misses; bounded buffer exhaustion; truly recoverable media metadata; safe ejection/power-loss energy and external-drive model identification.

**Document history**

- **2026-11-12:** Created Part 8 / Section 16, translating Sections 10–15 into a candidate STM32 implementation and timed proof plan. Incorporated the newly acquired Macintosh/external drive as a prospective measurement reference without assuming its condition or pinout. No MCU part number, circuit or flash transaction algorithm approved yet.
