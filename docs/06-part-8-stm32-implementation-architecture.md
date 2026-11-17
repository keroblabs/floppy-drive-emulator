# Sony MFD-51W Drive Interface Protocol — Part 8: ATF1508 + STM32F411 Implementation Architecture

**Document:** 06 — Part 8  
**Protocol section:** **16 — STM32 implementation architecture**  
**Status:** **Preferred development architecture: 5-V ATF1508-family CPLD + STM32F411**; exact suffixes/packages, CPLD fit, pinmux, signals, flash recovery and PCB not yet approved  
**Target:** Macintosh Plus M0001AP internal 20-pin Sony MFD-51W emulator  
**Revised:** 17 November 2026

> **Sequence:** [Document 02](02-macintosh-20-pin-interface.md) owns the internal connector; [Part 1 / Sections 1–9](06-part-1-drive-interface.md) drive control/status; [Part 2 / Section 10](06-part-2-serial-gcr-read.md) GCR read stream; [Part 3 / Section 11](06-part-3-media-architecture.md) sector records and cartridge; [Part 4 / Section 12](06-part-4-serial-gcr-write.md) WR capture and persistence; [Part 5 / Section 13](06-part-5-format-geometry-400k.md) standard reconstruction; [Part 6 / Section 14](06-part-6-error-handling-standard-media.md) faults; [Part 7 / Section 15](06-part-7-electrical-timing.md) electrical timing. **This Section 16 maps the fixed requirements to the preferred ATF1508 + STM32F411 hardware architecture.** Future Section 17 validates them.

**Labels:** **[DECISION]** settled project requirement; **[SOURCE]** published specification; **[PREFERRED]** selected design direction, not yet proven hardware; **[CANDIDATE]** an alternative; **[OPEN]** unresolved engineering gate. A candidate is not a finalized BOM, pinmux or PCB.

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

**[PREFERRED]** Divide responsibilities: the **ATF1508** is the deterministic Sony electrical/timing front end; the **STM32F411** owns structured track data, sector/format validation, SPI-slave DMA transmit, write input capture and removable NOR-flash persistence.

~~~text
Macintosh Plus IWM — internal MFD-51W 20-pin connector
  PH0/1/2 PH3 SEL /ENBL /WRREQ WR             RD pin 16
       |                                       ^
       v                                       |
     ATF1508 CPLD + 19.584-MHz reference -------+
     +-------------------------------------------+
     | ÷40 GCR cell clock = 489,600 Hz           |
     | SPI SCK/NSS control and MISO bit sampling |
     | RD pulse = 10 oscillator ticks per 1 bit  |
     | fast Sony status/RD selector and true OE  |
     | LSTRB/STEP/TACH status timing             |
     | MCU event and state/READY handshake       |
     +-----------------+-------------------------+
                       |  SPI SCK/NSS; F411 sends MISO
                       v
                STM32F411 (3.3 V)
     +-------------------------------------------+
     | SPI slave TX + circular DMA               |
     | ≥3 independent ≥12-KiB GCR track buffers |
     | TIM2/TIM5 both-edge WR capture proposal   |
     | track assembly and flash directory        |
     | separate SPI master for 2-MiB cartridge   |
     +-----------------+-------------------------+
                       |
                 SPI NOR cartridge
~~~

**[DECISION]** The IWM remains in the Macintosh. The STM32 sends *already encoded* GCR track bits; the CPLD does **not** perform 6-and-2 decoding or encoding. It converts serial 1 bits into timed RD pulses. The 4× expanded-SPI scheme is therefore unnecessary in this preferred architecture.

**[OPEN]** Confirm ATF1508 exact suffix, package/pin count and available macrocells before fixing the MCU-CPLD control bus or PCB. The whole design must pass a CPLD fitter and timing analysis.

### 16.3 Candidate STM32 and peripheral selection gate

**[PREFERRED]** **STM32F411xC/xE** replaces F407 and H723 as the leading MCU family. ST specifies a **100-MHz Cortex-M4F**, **128 KiB SRAM**, 256 KiB embedded Flash for xC or 512 KiB for xE, two 32-bit timers, two DMA controllers, and up to five SPI/I2S peripherals. Availability of individual peripheral pins depends on the exact package. The F411 has **no F407-style extra 64-KiB CCM RAM**.

Sources: [STM32F411 product family](https://www.st.com/en/microcontrollers-microprocessors/stm32f411.html), [STM32F411xC/E datasheet](https://www.st.com/content/ccc/resource/technical/document/datasheet/b3/a5/46/3b/b4/e5/4c/85/DM00115249.pdf/files/DM00115249.pdf/jcr%3Acontent/translations/en.DM00115249.pdf), [RM0383 reference manual](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf).

**[PREFERRED]** One external **19.584-MHz precision oscillator** clocks the CPLD. The CPLD divides by **40** to provide **489,600-Hz SPI SCK** to the F411 slave. A buffered voltage-compatible copy may also supply the F411 HSE bypass input, but the MCU's SPI-slave clock **does not depend on its PLL or SPI baud prescaler**.

**[CANDIDATE PLL illustration]** HSE 19.584 MHz, PLLM 12, PLLN 122, PLLP 2 would yield **99.552-MHz SYSCLK**, APB1 /2 **49.776 MHz**, and APB2 /1 **99.552 MHz**. Check the actual PLL constraints, power/flash latency and whether USB is needed before adopting; this illustration does not guarantee an exact USB 48-MHz clock.

**[OPEN]** Verify the actual part's complete top marking (CE/RE/VE or other variant and package), SPI-slave TX plus independent SPI NOR master pin groups, DMA streams, TIM2/TIM5 both-edge capture, SWD, CPLD control/status pins, and supply/reset. In a 48-pin package pin availability could be the decisive limit. **100 MHz and 128 KiB are feasibility constraints, not a demonstrated performance pass.**

### 16.4 Fast RD selector and electrical isolation

**[PREFERRED]** Use ATF1508 logic to select ROM-defined Sony status or GCR RD, respond promptly to PH0/PH1/PH2/SEL and qualify PH3/LSTRB control writes. The CPLD's /ENBL path must force **RD high impedance within 0.5 µs** independently of STM32 interrupts, firmware reset or flash operations.

**[DECISION]** Internal Macintosh Plus **pin 9 (−12 V)** and **pin 20 (PWM)** remain completely unconnected. RD is the single drive output at pin 16, with Section 15's output drive, thresholds, pulse width and true high-Z requirements.

**[OPEN]** The spare ATF1508 is reportedly a **5-V model**; confirm its full suffix and datasheet. A 5-V **core** does **not** mean direct CPLD outputs are safe for F411 3.3-V inputs. Select VCCIO and/or level translators, an RD tri-state output stage if needed, and verified power-up defaults. Prove all buffer propagation and RD loading at the actual host connector.

**[PREFERRED]** CPLD status registers should receive validated MCU state through a bounded handshake, avoiding glitches on READY, presence, protection, step feedback and selected track status. Do not allow a slow MCU poll to determine RD selector timing.

### 16.5 Serial GCR RD generation

**[PREFERRED]** Continuous **SPI-slave GCR transmission** is clocked by the CPLD. The F411 sends the full reconstructed standard track bitstream, from a DMA buffer, on SPI **MISO**. The CPLD supplies **SCK**, samples each bit at a proven phase and generates one narrow Sony RD pulse for every 1 bit; 0 produces no pulse.

| Parameter | Nominal value |
|---|---:|
| External precision reference | **19,584,000 Hz** |
| CPLD clock divider | **÷40** |
| SPI/GCR bit-cell clock | **489,600 Hz** |
| Bit-cell duration | **2.042484 µs** |
| RD pulse from 10 oscillator ticks | **0.510621 µs** |
| SPI-slave DMA data throughput | **61,200 bytes/s** |
| Example circular DMA TX buffer | **2 KiB**, two **1-KiB** halves |
| Available refill time per half | **16.732 ms** |

**[PREFERRED]** Start with MSB-first SPI slave and an experimentally determined CPOL/CPHA/NSS scheme (SPI mode 0 is only a candidate). The CPLD must **not start the serial clock before the first transmit byte is prepared**. Continuous ready-state clocking must not introduce repeated, missing or stale bytes on DMA half-transfer and transfer-complete events.

**[DECISION]** Adjacent 1 bits each produce a distinct pulse. The 10-tick pulse meets the **0.3–0.8-µs** Sony width requirement before output-stage distortion. Host RD selection and /ENBL tri-state are separate CPLD functions, not SPI output pins directly wired to the Macintosh.

**[DECISION]** Logical circular track address zero remains **sector 0**, with an independent live playback phase. A track bit length that is not byte-aligned requires software to **repack across the seam without inserting pad bits**. Readback immediately after formatting, phase coherence when switching tracks, and READY qualification are unchanged from Sections 12–15.

**[OPEN]** Prove first-bit setup/NSS behavior, F411 SPI underrun characteristics, seamless DMA refill, non-byte-aligned wrapping, CPLD MISO sampling phase, actual RD pulse timing and valid output source switching. The former F407 4× SPI-master scheme and timer-event engines are fallback alternatives, not the current preference.

### 16.6 WR both-edge capture and decoding

**[PREFERRED]** The STM32F411 timestamps **both rising and falling** WR data transitions using TIM2 or TIM5 (32-bit timers) and DMA. Gate state comes from qualified /WRREQ, with target cartridge, track, side and virtual RD phase latched for write decoding.

**[OPEN]** The exact capture channel and DMA routing must be verified in RM0383 and for the selected package. Do not assume a single capture channel always supports both polarities: two opposite-polarity channels may require an ordering/merge strategy. CPLD input qualification may help, but ordinary writes remain validated by STM32.

**[REFERENCE]** At a worst-case one transition per GCR cell, captures approach **489,600 events/s** = **1,958,400 bytes/s** for 32-bit timestamps. A **4,096-entry (16-KiB)** ring provides two 8-KiB halves, each **~4.183 ms** of worst-case service time. A 100-MHz MCU must drain and parse this continuously during a full-track format; this has not yet been measured.

~~~text
qualify write gate → latch target media and track/phase
  → DMA timestamps both WR edges
  → recover SYNC6 and GCR codeword alignment
  → ordinary sector update or >=50-SYNC6 format candidate
  → validate complete original encoded fields, checksums, IDs
  → publish coherent RAM generation for immediate readback
  → background SPI NOR persistence and safe metadata commit
~~~

**[DECISION]** Preserve unaffected sector records and address headers. Reject incomplete writes without falsely accepting them; treat interrupted formatting as permitted damage only to the target track, per Section 14.

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

**[PREFERRED]** The CPLD handles **qualified LSTRB rising edges**, fast Sony command feedback, and the **0.5–72-µs STEP low acknowledgment**. It notifies the STM32 of track, direction, motor and eject events through a registered interface, without decoding the Macintosh high-level disk format.

**[PREFERRED]** TACH generation is derived from the CPLD reference clock with five nominal target frequencies **394/429/472/525/590 Hz**. Its state must continue independently of the host momentarily selecting TACH; integer or fractional counters must be fitted and tested for period accuracy. GCR data-clock rate remains constant through all zones.

**[DECISION]** READY is a genuine handshake from STM32 to CPLD: only advertise ready with a qualified cartridge, correct track/side, valid DMA buffer and initial transmit byte, correct logical motor state and no conflicting write operation. No side-1 formatted sectors are invented for a 400K medium.

**[OPEN]** Prove the CPLD's macrocell use, status register update handshake, TACH counter width, selected output timing and reset defaults. Do not assume all these functions automatically fit the 128-macrocell device.

### 16.9 RAM and DMA budget

**[SOURCE]** The F411 contains **128 KiB SRAM**, and **does not have the F407's additional 64-KiB CCM bank**. Consequently the earlier 196-KiB H723 illustration and F407-specific CCM allocation are not applicable.

**[PREFERRED]** Keep the agreed **three independent complete ≥12-KiB track buffers** (36 KiB total). Allocate SPI slave DMA output, WR timestamp ring and flash transfers in verified DMA-accessible SRAM. No buffer feeding DMA may be modified until its consumer has released it.

**[CANDIDATE]** Illustrative **128-KiB** budget:

| Consumer | Initial allocation |
|---|---:|
| Three full cyclic track buffers | 36 KiB |
| WR input capture: 4,096 × 32-bit timestamps | 16 KiB |
| SPI slave TX: 2 × 1-KiB halves | 2 KiB |
| SPI NOR transfer staging | 8 KiB |
| Track rebuild and GCR record workspace | 8 KiB |
| Track directory/cache metadata | 8 KiB |
| Parser, firmware stacks and interrupts | 16 KiB |
| Additional tasks, diagnostics and workspace | 8 KiB |
| **Subtotal** | **102 KiB** |
| **Unallocated margin** | **26 KiB** |

**[OPEN]** This is not yet a proven linker map; evaluate worst-case parser scratch, flash transaction buffers, simultaneous dirty track generations, DMA stream conflicts, stack/heap use, bus contention and startup RAM. The nominal **61,200-byte/s** read stream is light, but WR peak-rate decoding and asynchronous NOR operations must pass timing tests.

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

**[PREFERRED]** Power the 5-V CPLD core from a suitably protected Macintosh **+5-V supply**, and the 3.3-V F411 and cartridge through a regulated supply. Choose CPLD VCCIO and any necessary translators after confirming the exact suffix. +12 V may remain unused but must not become a GPIO. **Pin 9 −12 V and pin 20 PWM are unconnected.** Provide RD power-off high-Z, brownout/reset supervisor and no back-power through ESD diodes or cartridge contacts. Selection of regulator, protection, output buffer and hold-up energy is deferred until worst-case current/rom-eject window analysis.

**[TEST RESOURCE]** A Macintosh **M0001AP with an included external drive** has been acquired and is expected soon. It will be useful for actual IWM/RD/WR/TACH measurements and comparison. **Inspect and identify both drives first**; an external DB-19 mechanism/revision must not be assumed to have the same internal 20-pin signal mapping, and advertised working condition is **not yet a verified bench result**.

### 16.13 Firmware ownership and priorities

**[PREFERRED]** Keep the CPLD's electrical work independent of the MCU's storage or parser scheduling.

| Owner | Responsibility | Must not depend on |
|---|---|---|
| ATF1508 | 19.584-MHz cell divider, SCK, short RD pulses, host status/RD mux, high-Z, LSTRB, STEP, TACH | CPU interrupts or flash |
| F411 SPI-slave + DMA | Continuous, correctly wrapped encoded GCR bytes | Blocking software and flash |
| F411 timer + DMA | Both-edge WR timestamps and gate timing | Flash programming |
| F411 bounded CPU work | WR parser, sector checks, track buffer updates, READY publication | Unbounded NOR busy |
| F411 background worker | SPI NOR reads, erase/program, journaling, power-loss recovery | Real-time pulse or capture path |
| External PC programmer | Logical disk-image import, offline 6-and-2 conversion and tag policy | Live MCU playback |

**[DECISION]** One worker owns the NOR bus. Every pending operation carries cartridge identity and generation. Pin active or dirty track generations; DMA may not read a buffer being modified by the parser. Keep logs asynchronous and do not invent extra Sony error or flow-control pins.

### 16.14 Development stages and acceptance gates

**[PREFERRED]** Incrementally prove feasibility before freezing the physical PCB:

| Stage | Experiment | Exit evidence |
|---|---|---|
| P0 | Identify exact F411 and ATF1508 suffixes/packages; safely inspect purchased Macintosh | Real pinout, device datasheets and viable programming tools |
| P1 | 19.584-MHz CPLD oscillator, ÷40, 10-tick RD pulse | Correct 489,600-Hz cells, 0.510621-µs pulse and stable timing |
| P2 | F411 SPI-slave TX circular DMA with CPLD SCK/NSS | No repeated/missed bits at DMA half/full boundaries or track seam |
| P3 | CPLD PH/SEL status mux, /ENBL tri-state, LSTRB/STEP and TACH | Required ≤0.5-µs RD response, command pulse/period conformity |
| P4 | F411 TIM2/TIM5 both-edge WR DMA while SPI TX runs | Full sector/format capture without overflow or pulse interference |
| P5 | Three full-track buffers and independent 2-MiB NOR | Host read timing survives concurrent cache/flash operations |
| P6 | Faults, repeated writes, write protection, cartridge removal and power cuts | No unrelated-track corruption; recoverable confirmed generations |
| P7 | Actual Macintosh Plus ROM reads/writes/formats/boots 400K and 800K | Measured and Macintosh-visible functional pass |
| P8 | Freeze pinmux, CPLD logic fit, power/voltage buffers and KiCad schematic | Evidence-backed electrical BOM and tested design |

**[OPEN]** The external drive supplied with the Macintosh is a reference for further measurement but is not assumed to use the internal 20-pin MFD-51W wiring.

### 16.15 Approved boundaries and pending choices

**Preferred development architecture [PREFERRED]:** 5-V ATF1508-family CPLD + **STM32F411** (full suffix/package pending) with **19.584-MHz precision oscillator**, **CPLD ÷40 serial clock = 489,600 Hz**, **10-tick = 0.510621-µs RD pulses**, F411 **SPI-slave circular DMA**, a separate SPI-NOR bus and timer/DMA **both-edge WR capture**. This supersedes H723-first and F407-first MCU candidate preference without approving a specific BOM.

**Settled existing media requirements remain unchanged:** standard 400K/800K only, structured **32+703-byte GCR sector records**, logical sector-0 circular buffer with independent RD phase, five TACH zones, **≥3 × 12-KiB RAM track buffers**, 2-MiB removable NOR, correct pins 9/20 isolation, genuine Macintosh ROM read/write/format and safe media errors.

**Open gates:** verify CPLD suffix/VCCIO and fitter resources, F411 package/pinmux/DMA; continuous first-byte-preloaded SPI-slave transmission and nonbyte-aligned cyclic wrap; power-on RD tri-state; WR capture CPU headroom at 100 MHz; real NOR flash-busy cache-miss, journal durability, safe eject and power integrity.

**Hardware precision alone does not automatically guarantee a lossless SPI-slave byte stream or safe handling of all media faults.**

**Document history**

- **2026-11-17:** Revised the preferred architecture to a 5-V ATF1508 + STM32F411, CPLD-controlled SPI-slave GCR at 489,600 Hz, 10-tick RD pulse shaping and 128-KiB MCU SRAM budget; superseded H723/F407-first preference without freezing component packages or PCB.

- **2026-11-12:** Created Part 8 / Section 16, translating Sections 10–15 into a candidate STM32 implementation and timed proof plan. Incorporated the newly acquired Macintosh/external drive as a prospective measurement reference without assuming its condition or pinout. No MCU part number, circuit or flash transaction algorithm approved yet.
