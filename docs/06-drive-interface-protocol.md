# Sony MFD-51W Drive Interface Protocol — Macintosh Plus

**Document:** 06 — Drive interface protocol  
**Status:** Scaffold established; **Section 4 (addressed status read) populated**; other operations pending detailed analysis  
**Target:** Macintosh Plus M0001AP, internal 20-pin floppy interface  
**Drive:** Sony MFD-51W 800K; common internal interface relevant to MFD-51W-03 compatibility  
**Implementation platform:** STM32 preferred; exact MCU, electrical buffers and timing architecture not yet chosen  
**Date:** 6 November 2026

> **Purpose:** Specify the *host-visible operations* of the Sony drive, not only its pinout. Check required operations against the Macintosh Plus ROM `.Sony` driver. A scaffold heading is a research plan, not an approved firmware requirement.
>
> **Agreed connector baseline:** Leave emulator pins **9 and 20 electrically unconnected**. This does not eliminate the need to inspect cable continuity, keying and orientation on a real computer before attaching the hardware.

## Table of contents

1. [Scope, conventions and boundaries](#1-scope-conventions-and-boundaries)
2. [References and evidence levels](#2-references-and-evidence-levels)
3. [Common host/drive interface model](#3-common-hostdrive-interface-model)
4. [Operation: addressed RD/status reading — detailed](#4-operation-addressed-rdstatus-reading--detailed)
   - [4.1 The RD path and enable](#41-the-rd-path-and-enable)
   - [4.2 Address convention and ROM conversion](#42-address-convention-and-rom-conversion)
   - [4.3 The complete 16-entry selection table](#43-the-complete-16-entry-selection-table)
   - [4.4 How the Macintosh reads status](#44-how-the-macintosh-reads-status)
   - [4.5 Status semantics and polarity](#45-status-semantics-and-polarity)
   - [4.6 Startup probes and undocumented entries](#46-startup-probes-and-undocumented-entries)
   - [4.7 STM32 contract and example](#47-stm32-contract-and-example)
   - [4.8 Remaining verification](#48-remaining-verification)
5. [Operation: drive selection, initialisation and disable](#5-operation-drive-selection-initialisation-and-disable)
6. [Operation: writing command latches with LSTRB](#6-operation-writing-command-latches-with-lstrb)
7. [Operation: seek, step, direction and recalibration](#7-operation-seek-step-direction-and-recalibration)
8. [Operation: motor, ready handshake and tachometer](#8-operation-motor-ready-handshake-and-tachometer)
9. [Operation: disk insertion, protection and eject](#9-operation-disk-insertion-protection-and-eject)
10. [Operation: serial GCR disk-data reading](#10-operation-serial-gcr-disk-data-reading)
11. [Operation: serial write-data capture](#11-operation-serial-write-data-capture)
12. [Operation: format, speed zones, sides and 400K compatibility](#12-operation-format-speed-zones-sides-and-400k-compatibility)
13. [Operation: error handling and unusual sequences](#13-operation-error-handling-and-unusual-sequences)
14. [Electrical requirements and timing](#14-electrical-requirements-and-timing)
15. [STM32 implementation architecture](#15-stm32-implementation-architecture)
16. [Validation and acceptance tests](#16-validation-and-acceptance-tests)
17. [Open questions and decision log](#17-open-questions-and-decision-log)

---

## 1. Scope, conventions and boundaries

The Macintosh retains its **MC68000, IWM, VIA and ROM**; we replace only the drive mechanism's host-visible electronics. A **sense/status read** selects one logical status value onto **RD (pin 16)** and reads it through the IWM status path. A **disk-data read** uses the same RD wire for a timed GCR data stream and the IWM data path. They must not be confused.

The **68000 addresses IWM memory-mapped registers**, not Sony selector numbers directly. The IWM/VIA then set the Sony input lines CA0, CA1, CA2 and SEL. This document assigns numeric labels to the resulting 16 physical combinations.

**In scope:** Internal Macintosh Plus 800K MFD-51W logical/electrical compatibility, with 400K disk-format compatibility to be studied. **Out of initial scope:** HD20, external-port daisy-chain, SuperDrive/MFM, Apple II floppy-controller emulation.

See [01 — Overview](01-overview.md), [02 — 20-pin electrical interface](02-macintosh-20-pin-interface.md), [03 — Sony drive revisions](03-mfd-51w-vs-mfd-51w-03.md).

## 2. References and evidence levels

1. **Apple**, *800K Double-Sided Drive Engineering Requirements Specification*, drawing 699-0452-A, September 1986, `3 (status selection, commands and timing): [scan](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_specs/669-0452-A_800K_Double-Sided_ERS_Sep86.pdf).
2. **Apple**, *Guide to the Macintosh Family Hardware*, 2nd ed., 1990, Chapter 9 (Macintosh host connector, IWM): [scan](https://vintageapple.org/inside_o/pdf/Guide_to_Macintosh_Family_Hardware_2nd_Edition_1990.pdf).
3. **Macintosh Plus ROM disassembly:** [`plus/resources/res_drvr_sony.s`](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s), especially `Sony_AdrDisk`, `Sony_AdrAndSense`, `Sony_Sense`, `CHKCONNECT` and `Sony_CkDCDExist`.
4. **Sony driver equates:** [`include/sonyequ.inc`](https://github.com/jonathanschilling/mac_rom/blob/main/include/sonyequ.inc), especially its drive register map and symbols around lines 278–308.
5. **Apple TIL #1605** (mechanism/cable compatibility): [scan](https://savagetaylor.com/TIL/TIL01605.pdf).

**[DOC]** hardware documentation; **[ROM]** observed in disassembled Plus driver; **[DECISION]** agreed emulator choice; **[OPEN]** unverified. The ROM is excellent evidence of *which addresses the Macintosh reads*, but cannot establish an original Sony output voltage or value for an undocumented selection.

## 3. Common host/drive interface model

| Internal pin | Sony / host signal | Emulator direction | Role |
|---:|---|---|---|
| 2 | PH0 / CA0 | Input | Selector bit |
| 4 | PH1 / CA1 | Input | Selector bit |
| 6 | PH2 / CA2 | Input | Selector bit or command value |
| 8 | PH3 / LSTRB | Input | Control-latch strobe |
| 12 | SEL | Input | Selector bit, driven via VIA |
| 14 | /ENBL.INT | Input | Active-low internal-drive enable |
| 16 | RD | Tri-state output | Sense status, data, or tach source |
| 10 | /WRREQ | Input | Write qualification/handshake; analyse later |
| 18 | WR | Input | Serial write data; analyse later |
| 9 | −12 V at Macintosh | **NC** | Never connect emulator electronics [DECISION] |
| 20 | PWM at Macintosh | **NC** | Never connect emulator electronics [DECISION] |

Ground and power pins are detailed in [Document 02](02-macintosh-20-pin-interface.md). Host-side **Q6 and Q7 are IWM controls, not extra pins sent to the drive**. The STM32 cannot observe an IWM Q6/Q7 change directly at this connector.

## 4. Operation: addressed RD/status reading — detailed

**Status:** Functional selection and ROM usage analysed; exact electrical timing and some revision-specific responses remain open.

### 4.1 The RD path and enable

**[DOC]** The Sony drive multiplexes selected status signals and head-specific serial read streams onto **RD, pin 16**. The address is CA0/CA1/CA2/SEL.

**/ENBL.INT = 0** enables the internal drive. While disabled, **RD must be high impedance**, not driven high or low. When enabled, RD conveys the selected source; a host sense read is made using the IWM's status mode and samples that wire as **SENSE in bit 7**.

There is no independent track-zero, disk-present, ready or write-protect output pin.

### 4.2 Address convention and ROM conversion

We retain the earlier project's **physical-signal address convention**:

~~~text
PROJECT_ADDR = bits [CA2 CA1 CA0 SEL]
             = (CA2 << 3) | (CA1 << 2) | (CA0 << 1) | SEL
~~~

The Plus ROM disassembly uses this **different numbering convention** for exactly the same four lines:

~~~text
ROM_ADDR     = bits [CA1 CA0 SEL CA2]
             = (CA1 << 3) | (CA0 << 2) | (SEL << 1) | CA2
~~~

**[ROM]** This is explicit in [`Sony_AdrDisk` (lines 1159–1194)](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1159-L1194). Beware: project `0xA` is **ROM address 5**, not ROM address 10.

### 4.3 The complete 16-entry selection table

| Project | ROM | CA2 | CA1 | CA0 | SEL | RD source | Sense meaning / caveat |
|---|---:|---:|---:|---:|---:|---|---|
| `0x0` | 0 | 0 | 0 | 0 | 0 | DIRTN | Direction feedback: 0 inward, 1 outward |
| `0x1` | 2 | 0 | 0 | 0 | 1 | /CSTIN | 0: disk in place |
| `0x2` | 4 | 0 | 0 | 1 | 0 | /STEP | 1: available for next step |
| `0x3` | 6 | 0 | 0 | 1 | 1 | /WRTPRT | 0: write-protected |
| `0x4` | 8 | 0 | 1 | 0 | 0 | /MOTORON | 0: motor commanded on |
| `0x5` | 10 | 0 | 1 | 0 | 1 | /TK0 | 0: at track 0 |
| `0x6` | 12 | 0 | 1 | 1 | 0 | EJECT / eject switch latch | Exact original internal MFD-51W readback **[OPEN]**; ROM equates mention SE |
| `0x7` | 14 | 0 | 1 | 1 | 1 | TACH | Dynamic tachometer feedback |
| `0x8` | 1 | 1 | 0 | 0 | 0 | RDDATA0 | Serial read stream, side/head 0 |
| `0x9` | 3 | 1 | 0 | 0 | 1 | RDDATA1 | Serial read stream, side/head 1 |
| `0xA` | 5 | 1 | 0 | 1 | 0 | **Unspecified for 800K** | ROM uses it for HD20 /Exist probing, not normal floppy sensing |
| `0xB` | 7 | 1 | 0 | 1 | 1 | **Unspecified for 800K** | ROM equates: “not used by these drivers” |
| `0xC` | 9 | 1 | 1 | 0 | 0 | SIDES / /SINGLE SIDE | 1: double-sided 800K mechanism |
| `0xD` | 11 | 1 | 1 | 0 | 1 | /READY | 0: drive ready |
| `0xE` | 13 | 1 | 1 | 1 | 0 | /DRVIN | 0: drive connected |
| `0xF` | 15 | 1 | 1 | 1 | 1 | NewIntf / REVISED | 1: newer interface supporting ready handshake |

Source cross-check: [`sonyequ.inc` register map](https://github.com/jonathanschilling/mac_rom/blob/main/include/sonyequ.inc#L278-L308).

**Important:** `0x8` and `0x9` select **timed GCR read-data waveforms**, not ordinary static status bits. Their signalling and the IWM data-read mode belong in Section 10. Nor should any number in the selector table be confused with a *68000 memory address*.

### 4.4 How the Macintosh reads status

**[ROM]** Actual code paths: [`Sony_AdrDisk`](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1159-L1194), [`Sony_AdrAndSense` and `Sony_Sense`](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1196-L1211).

1. **Select the drive.** Internal /ENBL.INT is active low. Only the selected drive may drive RD.
2. **Set the selector.** `Sony_AdrDisk` sets IWM PH0–PH2 (CA0–CA2) and VIA port A bit 5 (SEL) from the ROM-format address in D0. The ROM deliberately sets CA0 and CA1 high before changing the other selection signals, then clears those that must be low. Preserve this observation when studying potential address-switching hazards.
3. **Read IWM status.** `Sony_Sense` masks interrupts temporarily; accesses `q6H` (Q6=1) and reads `q7L` (Q7=0), obtaining the IWM status register. **Bit 7** reflects SENSE from the selected drive.
4. **Return to data mode.** `Sony_Sense` accesses `q6L` to restore Q6=0, then restores the interrupt state.

The above is a **host-side operation**; the emulator does not see Q6 or Q7 as pins. On the drive side, it simply responds to selection, enable and state with the correct RD output. **LSTRB is not pulsed to perform a sense read**; pulsing it writes a command latch.

### 4.5 Status semantics and polarity

The Plus driver tests the following levels:

| Selector | Selected signal | RD=0 | RD=1 | ROM evidence |
|---|---|---|---|---|
| `0x0` | DIRTN | Direction inward, higher track | Direction outward, toward track 0 | `DirLAdr` / `DirHAdr` |
| `0x1` | /CSTIN | Disk present | Disk absent | `DIPAdr` |
| `0x2` | /STEP | Step in progress | Step available | `StepLAdr` |
| `0x3` | /WRTPRT | Write-protected | Writable | `WrProtAdr` |
| `0x4` | /MOTORON | Motor on | Motor off | `MtrOnAdr` |
| `0x5` | /TK0 | At track 0 | Away from track 0 | `Tk0Adr` |
| `0xC` | SIDES | Single-sided drive | Double-sided drive | `SidesAdr` |
| `0xD` | /READY | Ready | Not ready | `ReadyAdr` |
| `0xE` | /DRVIN | Drive connected | No drive | `DrvExstAdr` |
| `0xF` | NewIntf | Legacy interface | New 800K-ready interface | `NewIntfAdr` |

**[OPEN]** Verify special cases (no cartridge, motor off, eject in progress) and the precise original MFD-51W readback at `0x6`. TACH is dynamic (the source references associate 60 pulses per revolution). DIRTN, TACH and eject timing still need electrical-source validation, not merely source-level interpretations.

### 4.6 Startup probes and undocumented entries

**[ROM]** During floppy discovery, `CHKCONNECT` ([lines 229–240](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L229-L240)) probes:

| ROM symbol (number) | Project address | Expected for our 800K emulator |
|---|---|---|
| `DrvExstAdr` (13) | `0xE` | **0:** installed |
| `SidesAdr` (9) | `0xC` | **1:** two-sided |
| `NewIntfAdr` (15) | `0xF` | **1:** ready-handshake interface |

These responses are central to how the Plus ROM identifies the device. `0xF` is **not** the mechanical revision of the Sony drive.

Two addresses are **not defined as ordinary 800K status outputs** in the examined documentation:

- **Project `0xA` = ROM 5.** `Sony_CkDCDExist` ([lines 1229–1243](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1229-L1243)) uses it as an **HD20 /Exist** test after HD20-specific identification conditions. This is not the usual internal MFD-51W floppy probe. It does *not* establish what a physical MFD-51W drives there.
- **Project `0xB` = ROM 7.** The equates say “not used by these drivers”; no standard Plus 800K read dependency was found.

**[DECISION]** Retain both addresses in the decoder model as **reserved, output level pending verification**. Do not invent SuperDrive or HD20 responses; do not assume unassigned means low, high or high impedance.

### 4.7 STM32 contract and example

**Provisional behavioural contract:**

1. **/ENBL.INT=1:** RD **high impedance**; status state can be retained as appropriate.
2. **/ENBL.INT=0:** use CA0/CA1/CA2/SEL to select RD source, with deterministic changeover meeting the electrical timing requirements still to be measured.
3. Static statuses return the correct sense level; **RDDATA0/RDDATA1** and **TACH** require their respective timed waveforms.
4. Only a valid **LSTRB command strobe**, not a status read, may modify command latch state.
5. The MCU must not expect to see the ROM's Q6/Q7 state on the connector.

Example with **cartridge inserted, writable, at track 0, motor on, logically ready, and internal drive enabled**:

| Project address | RD | Meaning |
|---|---:|---|
| `0x1` | 0 | Media present |
| `0x3` | 1 | Writable |
| `0x4` | 0 | Motor on |
| `0x5` | 0 | Track 0 |
| `0xC` | 1 | Two-sided |
| `0xD` | 0 | Ready |
| `0xE` | 0 | Drive installed |
| `0xF` | 1 | New 800K interface |

This demonstrates **logical status selection**, not the real-world delay after insertion or a motor-on command. We will not simulate mechanical delay unless the host requires a particular observable response sequence.

### 4.8 Remaining verification

- [ ] Confirm exact MFD-51W signals and polarity for **eject-readback, DIRTN, TACH**, and empty/no-disk cases from original electrical/timing documentation.
- [ ] Determine whether `0xA` and `0xB` need defined electrical return levels for any target software.
- [ ] Verify RD drive strength, transition latency, output-enable/disable timing, and the STM32 buffer approach.
- [ ] Investigate the ROM's CA0/CA1 preconditioning of SEL changes and any selector glitches.
- [ ] Verify status transitions for STEP, READY, MOTORON, CSTIN and eject against a real drive or instrumented ROM emulator.

## 5. Operation: drive selection, initialisation and disable

**TO DEVELOP.** ROM: `E_Sony_Open`, `Sony_GetDrive`, `Sony_DiskSelect` and IWM mode setup. Define reset defaults, drive enable qualification, RD tri-state, selector transitions, startup identification, and recovery when reenabled. External HD20 pass-through is reference-only.

## 6. Operation: writing command latches with LSTRB

**TO DEVELOP.** ROM: [`Sony_AdrAndStrb` / `Sony_Pulse`](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1213-L1227); equates `DirLAdr`, `DirHAdr`, `StepLAdr`, `MtrOnAdr`, `MtrOffAdr`, `EjectHAdr`. Provide full command/latch table: CA0/CA1/SEL select target, CA2 supplies value, LSTRB edge commits. Detail polarity, reset value, write timing, readback, and command acceptance while not ready.

## 7. Operation: seek, step, direction and recalibration

**TO DEVELOP.** ROM: `Sony_Seek`, `Sony_Recal`, `SONY_WAIT100`, `SONY_WAITREADY`. Define track range 0–79, direction, single-step triggering, /STEP feedback, recalibration to track zero, settle/ready sequencing, and track-boundary handling.

## 8. Operation: motor, ready handshake and tachometer

**TO DEVELOP.** ROM: `Sony_FastPower`, `Sony_PowerUp`, `SONY_WAITREADY`, `Sony_GetFreq1`, `Sony_WaitTach`. Define motor latch, tach waveform, /READY polling, motor-off behaviour, and original zone-speed expectations. Do **not** connect pin 20 PWM.

## 9. Operation: disk insertion, protection and eject

**TO DEVELOP.** ROM: `Sony_Check4Disk`, `Sony_CkDIP`, `Sony_Eject`. Specify inserted/empty states, write protection, latch/eject acknowledgement, disk removal, physical cartridge release handoff, and safe media-change handling.

## 10. Operation: serial GCR disk-data reading

**TO DEVELOP.** This is distinct from Section 4. Specify `RDDATA0`/`RDDATA1` selection, IWM data mode (Q6=Q7=0), bit-cell and transition timing, synchronisation, address/data fields, GCR decoding expectations, sector layout, and track/head changeover. ROM paths: `Sony_DiskPrime`, `Sony_RdAddr`, `Sony_RdData` and denibblisation routines.

## 11. Operation: serial write-data capture

**TO DEVELOP.** Specify WR (pin 18), /WRREQ (pin 10), valid write gating, IWM write/handshake modes, input-capture timing, serial GCR representation, buffering, flash commit, and write-protection enforcement. Inspect ROM write/format and handshake paths.

## 12. Operation: format, speed zones, sides and 400K compatibility

**TO DEVELOP.** ROM: `Sony_CtlFormat`, `Sony_DoFormat`, `Sony_FillInMarks` and GCR field routines. Document 80 tracks, zone-dependent sectors per track (12/11/10/9/8), both sides, format bytes, interleave, checksums, formatting and 400K disk-image compatibility. Distinguish a **400K-formatted disk** from emulating an original **400K-only drive**.

## 13. Operation: error handling and unusual sequences

**TO DEVELOP.** No disk, read/write failure, repeated step, out-of-range tracks, not-ready timeouts, eject during write, cartridge removal, write underrun, brownout/power loss, and recovery from unexpected host strobes.

## 14. Electrical requirements and timing

**TO DEVELOP.** Pin voltage thresholds, RD output buffers and high impedance, host input loading, status setup/hold/response limits, GCR pulse timing, 5 V to STM32 interfacing, power budget, fail-safe boot, cable continuity and pin-9/20 isolation. Link detailed connector/power tables to [Document 02](02-macintosh-20-pin-interface.md).

## 15. STM32 implementation architecture

**TO DEVELOP.** Selector decoder, LSTRB edge capture, status multiplexer, tri-state RD driver, timer/DMA GCR generation, capture of WR transitions, SPI flash cartridge buffering, interrupt priority, timing/jitter verification. STM32 is the selected *platform direction*, not yet a specific part number.

## 16. Validation and acceptance tests

**TO DEVELOP.** Build an evidence-backed checklist and test matrix covering:

- Drive enabled/disabled and RD high impedance.
- ROM startup probing (`0xE=0`, `0xC=1`, `0xF=1`).
- Address/ROM encoding conversion and all 16 selector combinations.
- CSTIN, WRTPRT, TK0, STEP, MOTORON, READY, eject transitions.
- Commands, seek and recalibration.
- Both heads, 400K/800K GCR read/write/format and boot.
- Electrical latency, no-disk state, power loss, cartridge removal.
- ROM trace and comparison to a physical Sony drive when available.

## 17. Open questions and decision log

### Agreed

- **[DECISION]** Internal Macintosh Plus M0001AP, 20-pin connector, MFD-51W behaviour first.
- **[DECISION]** STM32 preferred; exact MCU and 5 V interface not yet fixed.
- **[DECISION]** Pins **9 and 20** disconnected at emulator; no pin-9 hardware eject or pin-20 PWM.
- **[DECISION]** Software eject via standard command interface remains mandatory.
- **[DECISION]** Use Macintosh Plus `.Sony` driver as a reference for actual operations, alongside original Apple hardware documents.
- **[DECISION]** Required protocol timing yes; mechanically realistic delays only if needed for compatibility.

### Outstanding

1. Clarify original-drive responses at `0x6`, `0xA`, `0xB`.
2. Specify complete write-command table and state transitions.
3. Verify exact address/status and GCR timing for STM32 peripheral selection.
4. Verify harness orientation/continuity before connecting any prototype.
5. Reconcile historical documentation describing controller selection as open; STM32 is now preferred, without modifying other docs in this commit.

### Change history

- **2026-11-06:** Created Document 06: operation-first contents and a detailed, source-referenced **status-read** section. Correctly distinguished ROM address notation from project notation, identified ROM startup probes, and left unverified/read-unassigned outputs explicitly open. All subsequent operations remain scaffolded.
