# Sony MFD-51W Drive Interface Protocol — Part 1: Host Interface and Drive Operations

**Document:** 06 — Part 1 (Sections 1–9)  
**Status:** Sections **4 and 6–9 developed**; Section **5 scaffolded**; detailed timing/implementation open where marked  
**Target:** Macintosh Plus M0001AP, internal 20-pin floppy interface  
**Drive:** Sony MFD-51W 800K; common internal interface relevant to MFD-51W-03 compatibility  
**Implementation platform:** STM32 preferred; exact MCU, electrical buffers and timing architecture not yet chosen  
**Last updated:** 9 November 2026

> **Document organisation (9 November 2026):** This is **Part 1**, containing numbered Sections **1–9**. **[Part 2 / Section 10](06-part-2-serial-gcr-read.md)** develops serial reading. **[Part 3 / Section 11](06-part-3-media-architecture.md)** consolidates structured encoded-GCR sector storage plus RAW_TRACK compatibility, >=3 complete-track RAM buffers, global SYNC6 calibration, logical track metadata and TACH. **[Part 4 / Section 12](06-part-4-serial-gcr-write.md)** now develops serial writing; future sections remain **13–18**. These numbers do not imply approval of outstanding implementation details.
>
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
   - [6.1 Purpose and distinction from status reading](#61-purpose-and-the-essential-distinction-from-status-reading)
   - [6.2 Eight command-value encodings](#62-full-command-value-map--eight-combinations-four-functions)
   - [6.3 LSTRB write cycle and timing](#63-lstrb-write-cycle-sequence-and-required-timing)
   - [6.4 Macintosh Plus ROM command routine](#64-how-the-macintosh-plus-rom-actually-issues-commands)
   - [6.5 Control semantics and readback](#65-control-semantics-automatic-release-and-readback)
   - [6.6 800K software eject](#66-the-special-800k-eject-sequence)
   - [6.7 Disable and reset](#67-disable-reset-and-operation-lifecycle)
   - [6.8 STM32 decoder model](#68-preliminary-stm32-command-decoder-model)
   - [6.9 Tests and open items](#69-tests-boundaries-and-open-verification)
7. [Operation: seek, step, direction and recalibration](#7-operation-seek-step-direction-and-recalibration)
   - [7.1 Signals and selection](#71-relevant-signals-register-addresses-and-responsibilities)
   - [7.2 Track geometry](#72-track-geometry-and-virtual-position)
   - [7.3 ROM seek algorithm](#73-the-macintosh-plus-seek-algorithm--one-step-per-command)
   - [7.4 STEP acknowledgement](#74-step-acknowledgement-and-the-meaning-of-step)
   - [7.5 Electrical versus mechanical timing](#75-timing-electrical-handshake-versus-physical-mechanism-limits)
   - [7.6 READY, track zero, and power-on](#76-ready-tk0-disk-presence-and-power-on-initialization)
   - [7.7 Recalibration](#77-the-macintosh-plus-recalibration-home-to-track-zero-procedure)
   - [7.8 ROM polling and errors](#78-the-roms-post-seek-polling-and-error-behaviour)
   - [7.9 STM32 state model](#79-preliminary-stm32-state-and-event-model)
   - [7.10 Test matrix](#710-verification-matrix-and-remaining-questions)
8. [Operation: motor, ready handshake and tachometer](#8-operation-motor-ready-handshake-and-tachometer)
   - [8.1 Drive enable, motor and ready](#81-three-distinct-concepts-enabling-the-drive-commanding-the-spindle-and-being-ready)
   - [8.2 Motor command and shutdown](#82-motor-control-command-addresses-readback-and-shutdown)
   - [8.3 Five speed zones](#83-five-speed-zones-physical-800k-mechanism-versus-electronic-model)
   - [8.4 TACH waveform](#84-tach-is-a-selected-waveform-not-a-fixed-status-bit)
   - [8.5 ROM tach checks on 800K](#85-critical-rom-finding-800k-drives-can-still-be-tach-checked)
   - [8.6 How ROM measures tach](#86-how-the-plus-rom-measures-tachometer-speed)
   - [8.7 READY semantics and timing](#87-ready-definition-predicates-and-transitions)
   - [8.8 ROM startup and polling](#88-rom-motor-power-up-and-ready-polling)
   - [8.9 Insertion spin and disable](#89-automatic-insertion-spin-disabling-ejection-and-idle-tach)
   - [8.10 STM32 architecture](#810-proposed-stm32-timingready-architecture)
   - [8.11 Test matrix and open items](#811-verification-matrix-and-unresolved-details)
9. [Operation: disk insertion, protection and eject](#9-operation-disk-insertion-protection-and-eject)
   - [9.1 Interface signals and responsibilities](#91-interface-signals-selectors-and-software-responsibilities)
   - [9.2 Status polarity and empty-drive behaviour](#92-exact-sense-polarities-and-the-empty-drive-case)
   - [9.3 ROM insertion polling](#93-macintosh-plus-periodic-disk-insertion-detection)
   - [9.4 First access and protection](#94-first-disk-access-presence-recalibration-and-write-protection)
   - [9.5 Software eject sequence](#95-software-eject-command-and-required-host-visible-sequence)
   - [9.6 Logical versus physical eject](#96-logical-ejection-versus-physical-cartridge-release--a-necessary-new-state)
   - [9.7 Cartridge qualification](#97-insertion-qualification-and-cartridge-identity)
   - [9.8 Write-protect enforcement](#98-write-protect-sources-caching-and-enforcement)
   - [9.9 Flash removal and durability](#99-abrupt-removal-power-interruption-and-safe-flash-writes)
   - [9.10 STM32 lifecycle model](#910-preliminary-mediaeject-state-machine-and-outputs)
   - [9.11 Verification and decisions](#911-verification-matrix-and-open-decisions)
10. **[Serial GCR disk-data reading — Part 2](06-part-2-serial-gcr-read.md#10-operation-serial-gcr-disk-data-reading)** — developed in a separate document.
11. **[Shared media architecture — Part 3](06-part-3-media-architecture.md#11-shared-structured-gcrraw-track-media-timing-and-cartridge-architecture)** — developed.
12. **[Serial write-data capture — Part 4](06-part-4-serial-gcr-write.md#12-operation-serial-gcr-writing)** — developed (gate, WR edges, RAM capture, sector writes and format).
13–18. **[Future sections — roadmap](#continuation-and-future-parts)** — updated numbering.

- [Continuation and future parts](#continuation-and-future-parts)
- [Decisions and change history](#decisions-and-change-history)
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

**Status:** Command-address mapping, command acceptance edge and actual Macintosh Plus ROM calls established from original sources. Precise revision-specific output timing, exceptional sequences and STM32 latency margin remain subject to electrical verification.

### 6.1 Purpose and the essential distinction from status reading

**[DOC]** The Sony MFD-51W control interface has **four writable control functions**: **/DIRTN** (direction), **/STEP**, **/MOTORON** and **EJECT**. The host selects a command with **CA1, CA0 and SEL**, places the **new control value on CA2**, and sends a **low–high–low pulse on LSTRB** (PH3, connector pin 8). The drive captures the command at the **rising edge**, provided the drive is enabled. The subsequent falling edge ends the pulse; it is *not* the latch-write edge.

The same four CA/SEL inputs also choose the **RD/status source** described in Section 4. A specific four-bit combination can therefore have one meaning **during a status read** and a different meaning **when LSTRB is strobed**. Merely placing an address on the lines **must not** execute a command.

**[ROM]** This separation is implemented explicitly: `Sony_AdrAndSense` selects a register and reads the IWM SENSE/status path; `Sony_AdrAndStrb` selects a register and then invokes `Sony_Pulse` to strobe LSTRB.

**[DECISION]** The STM32 command decoder reacts to *qualified LSTRB rising edges*, not to CA/SEL changes alone. No dependency on a physical motor, head actuator or eject-solenoid implementation is implied.

### 6.2 Full command-value map — eight combinations, four functions

All four documented command functions have **SEL=0**. For these functions, **CA1:CA0** selects the latch and **CA2** sets its value. The table shows both the project's four-bit address convention (Section 4.2) and the **numeric address used by the Macintosh Plus ROM**. These are *command strobes*, not status reads.

| Function | CA1 | CA0 | SEL | CA2 | Project selector | ROM address / symbol | Interpretation of the strobe |
|---|---:|---:|---:|---:|---|---|---|
| **/DIRTN** | 0 | 0 | 0 | 0 | `0x0` | `0` / `DirLAdr` | Set stepping inward (higher track numbers) |
| **/DIRTN** | 0 | 0 | 0 | 1 | `0x8` | `1` / `DirHAdr` | Set stepping outward (toward track 0) |
| **/STEP** | 0 | 1 | 0 | 0 | `0x2` | `4` / `StepLAdr` | Assert low: request one step |
| **/STEP** | 0 | 1 | 0 | 1 | `0xA` | `5` / `StepHAdr` | Write high/released; *not* used for ordinary stepping by the Plus ROM |
| **/MOTORON** | 1 | 0 | 0 | 0 | `0x4` | `8` / `MtrOnAdr` | Turn motor on |
| **/MOTORON** | 1 | 0 | 0 | 1 | `0xC` | `9` / `MtrOffAdr` | Turn motor off |
| **EJECT** | 1 | 1 | 0 | 0 | `0x6` | `12` / `EjectLAdr` | Write low/inactive value; semantics during an active eject need confirmation |
| **EJECT** | 1 | 1 | 0 | 1 | `0xE` | `13` / `EjectHAdr` | Set EJECT high and begin the ejection operation |

**[DOC]** Hardware definitions: Apple 699-0321 (August 1985), ``3.1.2, 3.2.3–3.2.4; Apple 699-0452-A (September 1986), same sections. **[ROM]** Command symbols: [`sonyequ.inc` lines 278–308](https://github.com/jonathanschilling/mac_rom/blob/main/include/sonyequ.inc#L278-L308).

**Two important address collisions:**

- **Project `0xA`** is *unspecified as a normal 800K status read* (Section 4), **but a defined write** for `/STEP=1`. The ROM also uses the same physical selection as a special **HD20 /Exist sense probe**, but that is not ordinary 800K floppy status.
- **Project `0xE`** selects **/DRVIN** when read as a status, **but triggers EJECT** when its value is strobed. A status read of `0xE` must never eject media.

No control functions at **SEL=1** are defined for these four 800K commands. **[DECISION]** Do not invent latch writes for those combinations. The 1985/1986 specifications and the Plus ROM support the eight command encodings listed above.

### 6.3 LSTRB write-cycle sequence and required timing

**[DOC]** Host-to-drive write procedure:

1. **Enable the intended drive:** `/ENBL.INT = 0` (connector pin 14). A disabled drive must not accept a normal write as if enabled.
2. **Prepare the command:** set **CA0/CA1/SEL** to the target function and **CA2** to the desired value. Keep **LSTRB low** while the selector is being changed.
3. **Meet address setup:** hold CA0/CA1/CA2/SEL stable before the strobe.
4. **Assert LSTRB high:** the **rising edge** causes the drive to sample **CA2** into the selected control function (or to initiate the corresponding operation).
5. **Maintain the required high pulse width**, then bring **LSTRB low**.
6. Permit selector changes for the next operation only after completing the pulse; observe the safe **SEL-switching** sequence below.

Timing figures from the original manufacturer's command timing diagrams (printed **sheet 26** of Apple 699-0321, and **sheet 25** of Apple 699-0452-A):

| Figure parameter | 1985 699-0321 | 1986 699-0452-A | Interpretation / STM32 relevance |
|---|---|---|---|
| **T1** | **0.5 µs minimum** | **0.5 µs minimum** | Selector stable before LSTRB rises |
| **T2** | **1.0 µs minimum** | **1.0 µs minimum** | LSTRB-high pulse width |
| **T6** | **0.5 µs minimum** | **0.5 µs minimum** | Enable-to-address sequencing interval shown in the diagram |
| **T4** | **1.0 µs minimum** as printed | **1.0 µs maximum** as printed | Post-strobe latch-response interval; **specification discrepancy, do not silently choose one** |
| **T5** | **0.5 µs minimum** as printed | **0.5 µs maximum** as printed | Latch response around drive disable; **specification discrepancy** |

**[OPEN] Timing discrepancy:** The two diagrams print opposite **min/max** qualifications for **T4 and T5**. The 1986 diagram is a later revision, but this document does **not** claim that its limits automatically supersede those of the original MFD-51W. Confirm against the source hardware/electrical test plan before establishing timing-related MCU requirements. T1/T2/T6 are consistent in both documents.

The 800K **EJECT** cycle has additional constraints and is *not* simply an arbitrarily long general-purpose strobe; see `6.6.

### 6.4 How the Macintosh Plus ROM actually issues commands

**[ROM]** The relevant code is:

- [`Sony_AdrDisk`, lines 1159–1194](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1159-L1194): converts the **ROM address** in D0 into PH0/PH1/PH2 (CA0/CA1/CA2) and VIA **SEL** levels.
- [`Sony_AdrAndStrb` / `Sony_Pulse`, lines 1213–1227](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1213-L1227): writes the selected latch by pulsing PH3/LSTRB.
- [`Sony_Seek`, lines 1394–1417](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1394-L1417); [recalibration around lines 1613–1625](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1613-L1625): direction and step.
- [motor control around lines 1650–1678](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1650-L1678); [eject routine around lines 440–480](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L440-L480).

The ROM drives PH3 with two **IWM memory-mapped accesses** and uses two NOP instructions between them as a pulse-width delay:

~~~asm
; Conceptual excerpt from Sony_Pulse; A0 points to the IWM
    TST.B   ph3H(A0)     ; PH3/LSTRB rises -> drive samples command
    NOP
    NOP                 ; ensure adequate high width
    TST.B   ph3L(A0)     ; PH3/LSTRB falls -> finish the pulse
~~~

The routine temporarily masks processor interrupts around this pulse, preventing a host-side service routine from disturbing the command sequence. **There is no IWM command byte sent to the drive**; changing IWM phase-line register states *physically produces* CA0/CA1/CA2/LSTRB.

**Address-changing discipline:** `Sony_AdrDisk` first drives **CA0 and CA1 high**, then changes CA2, sets VIA **SEL**, and finally lowers CA0/CA1 to their requested levels. The source notes that **CA0=CA1=1 must be maintained while SEL changes from 1 to 0**. Preserve this observation in the protocol tests; the emulator must not inadvertently execute commands during intermediate selection states.

### 6.5 Control semantics, automatic release and readback

**/DIRTN — direction**

- **CA2=0:** inward, toward the disk centre / increasing track numbers.
- **CA2=1:** outward, toward track zero.
- **[DOC]** On drive disable (`/ENBL=1`), /DIRTN is preset to **0**.
- The original mechanism prohibits changing direction during head movement or settling. **[DECISION]** A virtual head does not need mechanical settling, but command ordering and valid step direction must remain coherent.
- Its feedback is selectable on RD at **project `0x0`** (Section 4).

**/STEP — one movement request, not a persistent “seek active” flag**

- A **low-going /STEP** command (CA2=0 at the accepted LSTRB rising edge) starts a one-track step according to the current /DIRTN.
- **[DOC]** The original drive accepts the falling /STEP event into its destination-track counter and then **releases /STEP to high automatically**. The Plus ROM waits for it to be high before issuing the next low-step strobe.
- `StepHAdr` (CA2=1) exists in the bit map, but normal seek/recalibration uses **`StepLAdr`** and the drive's automatic release. **Do not require an explicit STEP-high host write** for stepping to complete.
- `/STEP` status feedback is read at **project `0x2`**; distinguish this short acknowledgement from longer mechanical **/READY** behaviour.
- The virtual track update may be immediate once the step event is accepted, but the **/STEP feedback timing** needs validation (Sections 7 and 14).

**/MOTORON — spindle command**

- **CA2=0:** motor commanded **on**; **CA2=1:** motor **off**.
- **[DOC]** On `/ENBL=1`, /MOTORON is preset **high** (off).
- The Plus ROM checks `/MOTORON` status at **project `0x4`**, strobes `MtrOnAdr` only if needed, and also uses `MtrOffAdr` during power-down.
- The emulated motor is a *logical* state controlling readiness and data emission; it is not a requirement to spin a physical mechanism or wait for rotational inertia.

**EJECT — command-triggered operation, not merely a software-held level**

- **CA2=1** at the eject command strobe starts an eject operation; the original drive sets EJECT high on the **LSTRB rising edge**.
- **[DOC]** EJECT returns low when **/CSTIN rises** (disk no longer in place) or after an approximately **2-second maximum** timeout. The drive initializes EJECT low at power-on.
- The original eject mechanism can continue completing an initiated eject even if **/ENBL goes high**; a drive-disable transition is *not* proof that ejection should be aborted.
- `EjectLAdr` (CA2=0) is defined as a low command value, but the Plus ROM normally sends **`EjectHAdr`**, then allows the drive to terminate its operation. **[OPEN]** Whether an explicit low strobe has abort semantics during active eject has not been established; do not invent it.
- The RD selection at **project `0x6`** is labelled EJECT by Apple's 800K table, but original MFD-51W readback details remain under review in Section 4.

### 6.6 The special 800K eject sequence

The 800K mechanism has **software ejection through LSTRB** even with emulator connector **pin 9 unconnected**. The direct hardware eject signal on certain mechanism revisions is a different interface and is *not* needed for this command.

**[ROM]** The Plus driver's floppy eject routine performs the following sequence ([`SONY_CTL_EJECT` and `SONY_NEWDRVEJECT`, lines 440–480](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L440-L480)):

1. Invalidate the drive's cached sectors and command **motor off**.
2. Keep/reselect the floppy interface, then wait **200 ms**.
3. Select **`EjectHAdr`** = ROM address 13 = **project `0xE`**: CA1=1, CA0=1, SEL=0, CA2=1.
4. Bring **LSTRB high** to start eject.
5. For the **new-interface 800K drive**, bring **LSTRB low promptly** to finish a **short pulse**. The old-interface 400K path instead keeps LSTRB high for approximately **0.75 s**.
6. Poll **/CSTIN** (project `0x1`) until it becomes **1** (disk out), using 100-ms waits and a bounded retry count (up to approximately **1.6 s** of polling), then wait another **1 ms**. The driver subsequently applies its own disk-present suppression interval.

**[DOC]** Apple's eject timing diagram (699-0321 printed sheet 33; 699-0452-A printed sheet 32) specifies approximately:

| Quantity | Source requirement | Relevance |
|---|---|---|
| Motor-off before eject request | **≥200 ms** | The ROM explicitly waits 200 ms |
| LSTRB-high eject pulse | **≥1.0 µs, ≤300 ms** | The Plus uses the short 800K pulse, *not* 400K's long pulse |
| /CSTIN transition after eject request | **≤1.5 s** | Mechanical operation limit, not a mandatory artificial STM32 delay |
| Drive remains enabled after /CSTIN goes high | **≥150 µs** | The ROM waits at least another 1 ms after detecting no disk |
| EJECT internal active-state timeout | **≤2 s** (separate functional description) | Bound the command state if no eject completion is observed |

**[DECISION]** For the STM32, distinguish **command accepted**, **logical media removed**, and **physical cartridge released**. The user-visible cartridge mechanism may be much simpler than Sony's, but the Macintosh must receive coherent **/CSTIN** and (where relevant) EJECT transitions. Do not *force* 200 ms of MCU-internal motor simulation; the **host** already performs that pre-eject wait.

### 6.7 Disable, reset and operation lifecycle

The original specifications state that **/ENBL high** disables communication, places RD in **high impedance**, and presets control latches to documented inactive defaults. At least:

- **/DIRTN = 0** (inward direction).
- **/MOTORON = 1** (motor off).
- **EJECT = 0** at *power-on*; an **ongoing eject** can continue across interface disable.
- **/STEP** is normally released high after an accepted request. Its precise disable/reset pulse behaviour remains **[OPEN]**; avoid manufacturing a step on re-enable.

**[DECISION]** Do not reset the *virtual track number*, mounted media state, or persistent flash state merely because /ENBL is deasserted. Those are different from the transient command latches. Likewise, no deferred command should be spuriously executed when the drive is re-enabled if LSTRB was already high.

These reset/lifecycle rules should be reconciled with Section 5 when that section is developed.

### 6.8 Preliminary STM32 command-decoder model

This is **pseudocode for the observable interface**, not a polling-loop or interrupt-latency design:

~~~text
On /ENBL changing inactive (HIGH):
    tri_state(RD)
    direction = INWARD                  # documented preset
    motor_commanded = OFF               # documented preset
    release_step_output_if_appropriate() # verify precise STEP timing
    # An eject already in progress is NOT necessarily cancelled.
    # Do not reset virtual track/media merely because enable is inactive.

On LSTRB rising:
    if /ENBL is not LOW:
        ignore command
    else:
        atomically_sample(CA1, CA0, SEL, CA2)
        if SEL != 0:
            no documented MFD-51W command: do not invent one
        else if CA1:CA0 == 00:          # DIRTN
            direction = (CA2 == 0) ? INWARD : OUTWARD
        else if CA1:CA0 == 01:          # STEP
            if CA2 == 0:
                accept_one_step(direction)
                step_feedback = LOW, then release to HIGH
            else:
                step_feedback = HIGH    # precise hardware effects open
        else if CA1:CA0 == 10:          # MOTORON
            motor_commanded = (CA2 == 0) ? ON : OFF
        else if CA1:CA0 == 11:          # EJECT
            if CA2 == 1:
                start_eject_operation()
            else:
                low/inactive command value; no assumed abort semantics

On LSTRB falling:
    end strobe pulse; DO NOT write command a second time

At any time when /ENBL is LOW:
    RD = read_selected_source(CA2, CA1, CA0, SEL)
    # Status/data mux is separate from the LSTRB command decoder.
~~~

**Architecture caution:** ROM pulses are microseconds long. The final STM32 implementation must capture **LSTRB edges** and handle RD/status changes deterministically; a slow, occasionally scheduled polling loop is not an approved implementation. Timer/input-capture, external logic or interrupt-assisted buffering are candidates for Section 16.

### 6.9 Tests, boundaries and open verification

| Test | Expected host-visible observation | Evidence |
|---|---|---|
| Change CA0/CA1/CA2/SEL, no LSTRB pulse | RD selection may change; **no control command executes** | [DOC] |
| Strobe DIRTN=0 / DIRTN=1 | Direction feedback and subsequent step direction follow requested value | [DOC]/[ROM] |
| Strobe STEP=0 once | Exactly **one** virtual step, then /STEP returns high | [DOC]/[ROM] |
| Strobe STEP=1 | No *additional* step generated | [ROM], other semantics [OPEN] |
| Strobe MOTORON=0 / MOTORON=1 | Motor feedback becomes on/off | [DOC]/[ROM] |
| Strobe EJECT=1 | Eject operation initiated; /CSTIN becomes absent when logically removed | [DOC]/[ROM] |
| Read project `0xE` without stroking | /DRVIN sense read; **never triggers eject** | [DOC] |
| Strobe project `0xA` | STEP-high write encoding; **not** a valid ordinary 800K read status | [DOC]/[ROM] |
| Hold LSTRB high without another rising edge | No repeated step or multiple eject requests | [DOC] |
| /ENBL inactive | Commands ignored; RD high impedance; default latches; pending eject completion respected | [DOC], exact edge cases [OPEN] |
| Drive disable/re-enable | Track/media persisted, no phantom step/eject | [DECISION] |

**Still open before firmware is final:**

1. Reconcile 1985 versus 1986 **T4/T5 min/max** discrepancies; verify command-edge latency requirements and robustness to selector transients.
2. Establish exact **/STEP low duration**, acknowledgement timing and permissible re-strobe cadence, plus /READY behaviour (Section 7).
3. Verify the original **EJECT low strobe** behaviour and any required readback semantics at project `0x6`.
4. Decide whether a hardware pulse-capture/command-state front end is necessary for guaranteed STM32 response at the final CPU clock and interrupt load.
5. Validate enable/high-impedance timing and pending-eject behaviour against an actual MFD-51W/-03 installation if possible.

## 7. Operation: seek, step, direction and recalibration

**Status:** Core algorithm, command/status mapping, ROM use and original drive timing verified. Emulator edge-case policy and physical-hardware conformance remain open.

**Design principle:** The STM32 maintains a **virtual track position**. There is no stepper motor and no mechanically realistic seek/settle delay. We nevertheless preserve the *interface contract*: one accepted step per valid command, appropriate `/STEP` feedback, `/TK0` at the zero boundary, and `/READY` only when the selected data stream can actually be served.

### 7.1 Relevant signals, register addresses and responsibilities

This operation is implemented entirely with the shared interface from Sections 3–6. No separate seek, track-zero or step connector pins exist.

| Function | Physical line / RD selection | Project selector | ROM address / symbol | Behaviour |
|---|---|---|---|---|
| Write direction **inward** | LSTRB strobe; CA1=0, CA0=0, SEL=0, CA2=0 | `0x0` | `0 / DirLAdr` | Subsequent step goes to greater track numbers |
| Write direction **outward** | LSTRB strobe; same latch, CA2=1 | `0x8` | `1 / DirHAdr` | Subsequent step goes toward track zero |
| Request **one step** | LSTRB strobe; CA1=0, CA0=1, SEL=0, CA2=0 | `0x2` | `4 / StepLAdr` | Drive accepts a low-going `/STEP` event |
| Read **step acknowledgement** | Selected on RD pin 16 (no strobe) | `0x2` | `4 / StepLAdr` | `1` = available for another step; `0` = still processing |
| Read **track zero** | Selected on RD pin 16 (no strobe) | `0x5` | `10 / Tk0Adr` | `0` = at track zero |
| Read **ready** | Selected on RD pin 16 (no strobe) | `0xD` | `11 / ReadyAdr` | `0` = data transfer permitted |
| Select/enable drive | `/ENBL.INT` pin 14 | — | IWM internal-drive enable | Qualifies command/status communication |

**[DOC]** `/DIRTN` is active low for inward movement; the original drive sets this latch to **0 on disable**. `/STEP` is active low and *automatically returns high after a request has been registered*. **[ROM]** The normal Plus code uses `StepLAdr` to request a step; it does not need a separate `StepHAdr` strobe to complete each one.

### 7.2 Track geometry and virtual position

**[DOC]** The 800K mechanism uses **80 concentric tracks per side**, numbered **0–79**, with **track 0 outermost** and greater track numbers closer to the centre. A double-sided disk has **two heads/sides but one common radial track position**. Changing head/side via the data selection does *not* itself cause a radial step.

**[DECISION]** Model physical head position as one integer `track` independent of mounted-media contents and current side. A normal step updates exactly one track in the current direction:

~~~text
direction = INWARD  (DIRTN=0)  => track increases by 1
direction = OUTWARD (DIRTN=1)  => track decreases by 1

track=0  => /TK0=0
track>0  => /TK0=1
~~~

The **preferred safe emulator policy** at the ends is to **saturate**, not wrap: outward steps while at 0 remain at 0; inward steps while at 79 remain at 79. **[OPEN]** Verify end-stop behaviour and whether a more exact MFD-51W response is needed for exotic software. Never allow an invalid index to address SPI flash beyond the 80-track layout.

Disk insertion or removal does not inherently change the head's current radial coordinate. A **power-on initialization**, however, is a separate event (see `7.6).

Track-zone transitions (15→16, 31→32, 47→48, 63→64 and reverse) affect GCR/rotation-generation parameters and buffered track selection, **not the count of steps**; the timing/data-engine interaction is covered in Sections 8, 10 and 12.

### 7.3 The Macintosh Plus seek algorithm — one step per command

**[ROM]** [`E_Sony_Seek / Sony_Seek`, lines 1365–1432](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1365-L1432).

The ROM is given a desired track in `D6`; **bit 11 is the selected side and is cleared before radial seeking**. It first calls `Sony_SetSpeed` for the *target track zone* (new-interface/800K drives use the self-regulated speed path).

For the ordinary case where the driver's stored track is known:

1. Read the driver's software `Track` variable (`D7`). If it equals the target, there is **no step**.
2. Compare current and target tracks.
3. If current < target, select **`DirLAdr`** (inward, CA2=0) and a `+1` track delta; otherwise select **`DirHAdr`** (outward, CA2=1) and a `−1` delta.
4. **Strobe the direction command** through `Sony_AdrAndStrb` (Section 6).
5. **Read `/STEP`** via `StepLAdr`. If it is low, return a step-related error; otherwise continue.
6. With the already-selected `StepLAdr` address, **pulse LSTRB** through `Sony_Pulse`. This requests exactly **one radial step**.
7. For an **800K/new-interface drive**, `SONY_WAIT100` polls `/STEP` until it returns high (bounded; see `7.5). For a legacy 400K drive, the branch uses a fixed wait instead.
8. Repeat until the desired track is reached.
9. At the end, take the **new-interface `/READY` path**, not an old 400K mechanical head-settle delay; see `7.5.

**Important:** The ROM stores its *expected* next track in its own variables before it issues the step command; that is a **software prediction**, not a report that the drive has physically arrived. The emulator must ensure that every accepted pulse results in one coherent virtual-track update. If a host step cannot be accepted, it must not silently advance the internal track.

If the ROM considers its stored position *unknown* (negative `Track`), it calls recalibration first; the exact retry flow and caller-level destination handling should be tested rather than assuming a full recalibration-and-seek sequence is always performed in this one routine.

### 7.4 Step acknowledgement and the meaning of `/STEP`

**[DOC]** The original drive treats the falling transition of `/STEP` as the accepted head-position-counter increment/decrement. It then **automatically restores `/STEP` high** as acknowledgement. The host polls the RD mux at project `0x2`; `/STEP=1` tells it that another step may be requested.

Consequences:

- A single accepted **LSTRB rising edge** with `CA1=0, CA0=1, SEL=0, CA2=0` produces **one** step request; continued LSTRB high does not repeat it.
- The host does **not** hold the physical `/STEP` line low externally. The host *writes the drive command*; the drive's **internal `/STEP` state** briefly becomes low, then returns high.
- The **`StepHAdr`** (project `0xA`) encoding is a possible high-value write, **not** something required after each normal step.
- `/STEP` must not be used as a synonym for `/READY`. The former means the request was accepted/released; the latter controls the host's subsequent data access.

**[DECISION]** A virtual track update may occur **immediately upon acceptance** of a valid step; however, its **feedback waveform** must satisfy the interface's pulse characteristics. A hardware timer or other deterministic device may be needed for the brief low-to-high acknowledgement. Do not preserve a millisecond “motor movement” delay just to make stepping appear mechanical.

**[OPEN]** Confirm whether an extremely brief low period must actually be externally observable for *all* Plus ROM polling paths. For initial conformance, use a bounded pulse respecting the documented original-drive low pulse width rather than forcing `/STEP=1` permanently.

### 7.5 Timing: electrical handshake versus physical-mechanism limits

**[DOC]** Both original **699-0321** (1985, head-access timing figure, printed sheet 27) and **699-0452-A** (1986, `3.4.3, printed sheets 26–27, and `3.4.6, printed sheet 31) provide the following characteristics. The two revisions agree on the principal values.

| Parameter | Original-drive requirement/limit | Meaning for our emulator |
|---|---|---|
| /ENBL active before changing DIRTN (figure T6) | **≥0.5 µs** | Observe selection/setup; host ordinarily provides it |
| Direction established before STEP (T1) | **≥1.0 µs** | Honour host command order; never let two strobes collapse into one |
| **/STEP low pulse width** (T2) | **0.5–72 µs** | Real *interface feedback* pulse; do not model milliseconds of physical travel |
| Additional direction/step separations (T3–T5) | **≥0.5 µs each** | Physical command/pulse spacing constraints; verify with analyser |
| Write gate off before beginning a STEP (`3.4.3.2) | **≥620 µs** | Prevent head change across a live write; cross-check write-path integration |
| /READY becomes not-ready following STEP (`3.4.3.3) | **≤150 µs** | Original drive invalidates readiness when head movement begins |
| /READY after one step in same speed block | **≤36 ms** | Original physical settling ceiling; **not** a mandatory firmware delay |
| /READY after one step with speed-block change | **≤152 ms** | Includes original speed adjustment; **not** a mandatory firmware delay |
| /READY after maximal-rate run of steps | **≤600 ms** | Original mechanism worst-case upper bound; **not** a mandatory firmware delay |
| /TK0 state transition after crossing track-zero boundary (`3.4.6) | **≤6 ms** | STM32 can update logically much sooner |

**Interpretation:** The **minimum** spacing/pulse requirements are real protocol constraints. The large **maximum** times primarily describe how slowly a *mechanical* drive may complete a movement; they do **not** require the emulator to delay for 36/152/600 ms. It is acceptable in principle to present stable /READY sooner **if the requested track's timed GCR stream is already prepared and host-visible ordering remains correct**.

**[DECISION]** Avoid a fixed “seek duration” in firmware. The emulator must instead be ready for the next command/status read as soon as its logical state and any required output buffers are coherent.

**[OPEN]** The source's `/READY` waveform and exact minimum observed transition time should be validated in tests; the diagram specifies an upper bound, not a minimum time for remaining not-ready. If the host assumes a detectable busy phase in an untested code path, measure and accommodate that demonstrable requirement rather than inventing a delay pre-emptively.

### 7.6 /READY, /TK0, disk presence, and power-on initialization

**/TK0 is a head-position report, not media identification.**

**[DOC]** The original `/TK0` status is **low** when the head is on track 0 *or farther outward than track 0*. The hardware description notes that it is **latched** and updated by the physical track-zero sensor **only when the mechanism is not in power-save mode**. Thus it is not simply a continuously live sensor output under all power states.

**[DECISION]** For the initial emulator, present `/TK0=0` iff virtual `track=0` while the drive is selected. Keep the virtual track on ordinary enable/disable transitions; do not reset it merely because `/ENBL` is high. More exact latch/power-save nuances are a validation item if required by software.

**/READY is different from step acknowledgement.**

**[DOC]** Original `/READY=0` means the mechanism can transfer data: disk in place, requested head position settled, and motor at the required speed. **[DECISION]** For the STM32 this maps to *real logical readiness*: cartridge present, no write/media-transition fault, motor logically on, and the appropriate track/head data ready for the timed bitstream engine. A virtual track change must invalidate any **stale** stream and select/reconfigure the new track before advertising readiness. Never return `/READY=0` merely because `/STEP` has released if storage preparation is still incomplete.

**Power-on baseline:**

**[DOC]** Sony specifies that the head is **automatically taken to track 0 on power-up** (`3.5.3, both revisions). Therefore, **[DECISION]** initial `track=0` is the right electronic equivalent. This says nothing about whether the cartridge is inserted or the spindle logically running.

**Power-down/disable:**

**[DOC]** On `/ENBL` high, `/DIRTN` is preset to 0 and `/MOTORON` to off, while the physical-drive mechanism may still complete an in-progress operation. **[DECISION]** Preserve virtual head position across enable disable; on re-enable, there must be no phantom step or recalibration unless commanded. Preserve persistent media state independently of drive-enable.

**[OPEN]** Verify exact response to power interruption during a pending write, especially whether the logical track should be reinitialised at MCU restart (power-on) independently of media-content recovery.

### 7.7 The Macintosh Plus recalibration (home-to-track-zero) procedure

**[ROM]** [`E_Sony_Recal / Sony_Recal`, lines 1558–1636](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1558-L1636).

The ROM's recalibration routine is not a special hardware command. It is a **sequence of ordinary status reads and direction/step commands**:

1. Invalidate the cached track data.
2. Ensure that the IWM is in its intended mode, and power/select the drive as needed. The ROM may program the IWM mode register and then power the drive up.
3. Initialise a bounded **track-zero search** (source starts its counter at `0x50` = 80).
4. On each iteration, wait the driver's nominal inter-step interval: **12 ms for old-interface 400K**, but **6 ms for new-interface 800K** (the ROM halves `SeekTime`).
5. Read **`/TK0`**. If **low**, recalibration succeeds and the driver stores `Track=0`.
6. Otherwise strobe **DIRTN outward** (`DirHAdr`), check **`/STEP`** is **high**, then issue a **STEP-low strobe** (`StepLAdr`).
7. Repeat until track zero is found or the bounded counter expires.

An **initial zero track** makes this procedure finish at the first `/TK0` check, but even if the virtual track is 79, repeated outward steps will eventually reach zero.

**[ROM]** If `/STEP` is low when the routine wants another step, it reports **`cantStepErr = −75`**. If the counter expires without `/TK0` becoming low, it reports **`tk0BadErr = −76`**. An IWM setup failure is separately reported as **`initIWMErr = −77`**. These errors are **ROM-side outcomes**, not error bytes sent over the Sony connector.

The source's decrement/test occurs before issuing a step at the exhaustion boundary; treat **80 as the nominal search bound**, not a promise that precisely 80 hardware pulses always occur.

### 7.8 The ROM's post-seek polling and error behaviour

**[ROM]** [`SONY_WAIT100`, lines 1434–1445](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1434-L1445) differs by interface generation:

- **Old-interface 400K mechanism:** fixed `SeekTime` (12 ms) via the ROM's delay path.
- **New-interface 800K mechanism:** poll **`/STEP`** repeatedly, with an initial loop counter of `0x50` (up to about 81 iterations). This is a **poll count**, not a promise of exactly 81 µs.
- The next iteration again checks `/STEP=1` before sending a new step. A device that leaves `/STEP` low indefinitely can fail with `cantStepErr`.

After seeking, [`SONY_WAITREADY`, lines 1466–1485](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1466-L1485) checks **`/READY`** for a **new-interface 800K drive**:

- For the **post-seek** path, the ROM first waits **500 µs**, then polls `/READY`.
- It polls at roughly **1 ms** intervals, using a counter initialized for approximately **1 second** of polling.
- **Crucial:** on timeout the ROM **continues anyway**; it does not itself report a `/READY` timeout error at this point. A device stuck not-ready may consequently cause downstream read/write failures.
- For the **old-interface 400K path**, it uses the previously specified fixed waiting interval instead.

**[DECISION]** We must supply **correct /STEP and /READY statuses**, not artificially reproduce either the ROM's built-in delays (the host already waits) or the Sony mechanism's additional mechanical time. In particular, **responding ready early is not forbidden by the cited maximum-only head-settle timing**, provided actual read/write operations are serviceable.

### 7.9 Preliminary STM32 state and event model

The following is a **functional contract**, not a choice of interrupt, DMA or timer implementation.

~~~text
State:
    track: integer 0..79                # one radial position shared by both sides
    direction: INWARD / OUTWARD
    step_feedback: HIGH / LOW
    motor_commanded: ON / OFF
    cartridge_present: bool
    track_stream_ready: bool             # prepared/selected data for current track
    write_active: bool
    ready: derived logical status

On MCU power-on (after hardware-safe initialization):
    track = 0                            # original Sony power-on homes to zero
    direction = INWARD                  # /DIRTN preset low
    step_feedback = HIGH
    motor_commanded = OFF
    track_stream_ready = false          # until data preparation is complete

On qualified DIRTN write:
    direction = (CA2 == 0) ? INWARD : OUTWARD

On qualified STEP-low write (one valid LSTRB rising edge):
    if step_feedback is HIGH and acceptance conditions hold:
        step_feedback = LOW
        next_track = clamp(track + (direction == INWARD ? +1 : -1), 0, 79)
        track = next_track
        invalidate_or_switch_track_stream(track)
        schedule_step_feedback_return_to_HIGH()   # bounded electrical pulse
    else:
        do_not_create_an_extra_step()              # exceptional case policy pending

On the scheduled feedback release:
    step_feedback = HIGH

RD status mux whenever enabled:
    project 0x2 -> step_feedback
    project 0x5 -> (track == 0) ? LOW : HIGH
    project 0xD -> ready ? LOW : HIGH

ready is true only if:
    cartridge_present
    and motor_commanded == ON
    and selected track/head data can be served correctly
    and there is no conflicting write/media transition

On normal drive disable (/ENBL rising):
    tri_state(RD)
    direction = INWARD
    motor_commanded = OFF
    retain track and cartridge state
    do_not_generate_step()
~~~

**Technical refinement:** The status mux reads shared state that changes in the command decoder. The actual STM32 design must publish a **coherent track/status/data-stream state**, avoiding an instant when `/TK0` reflects a new track but a stale track's GCR bitstream is still emitted. A track-cache switch may occur immediately if the required data is resident in memory; otherwise set `/READY` to not-ready until it is prepared. We need no artificial motion timer.

**Out-of-range and unexpected-input policy remains provisional:** Clamp at 0/79 for safety; do not interpret head/side changes as track steps; do not advance during a disallowed write/transition. The exact behaviour after a step request during `/WRREQ` asserted, rapid re-strobe, cartridge removal and simultaneous state transitions needs test coverage and Section 12 write-path integration.

### 7.10 Verification matrix and remaining questions

| Scenario | Expected result | Status |
|---|---|---|
| MCU power-on, no cartridge | `track=0`; `/TK0=0` when enabled; `/READY=1` | [DECISION] based on [DOC] power-on homing |
| Strobe DIRTN inward, then STEP low | Track increases exactly once; `/STEP` acknowledges and returns high | [DOC]/[ROM] |
| Strobe DIRTN outward, then STEP low | Track decreases exactly once | [DOC]/[ROM] |
| Hold LSTRB high after STEP | No multiple steps | [DOC] |
| Attempt step at track 0 outward | Track does not wrap below 0; `/TK0=0` | [DECISION], end-stop fidelity [OPEN] |
| Attempt step at track 79 inward | Track does not wrap above 79; safe image bounds | [DECISION], end-stop fidelity [OPEN] |
| Seek across zones 15→16, etc. | Exactly one step; stream/speed-zone metadata switched before `/READY=0` | [DECISION] |
| Read `/STEP` while low/high | Correct feedback, not mistaken for `/READY` | [DOC]/[ROM] |
| Recalibrate from virtual track 79 | Repeated outward steps; eventually `/TK0=0`; ROM stores track 0 | [ROM] |
| Recalibrate already at 0 | First `/TK0` read succeeds; no unnecessary step | [ROM] |
| Leave `/STEP` low forever (negative test) | ROM cannot continue normal seeks; expected `cantStepErr` path | [ROM] |
| Leave `/TK0` high forever (negative test) | ROM bounded recalibration fails with `tk0BadErr` | [ROM] |
| Interrupt stream preparation with a step | `/READY` reflects real ability to supply new track; never serves wrong track as ready | [DECISION] |
| Disable and re-enable without MCU power loss | Virtual track retained; no phantom step | [DECISION] |
| Change SEL/head without stepping | Same track, different side's data; no radial movement | [DOC] |

**Open work before acceptance:**

1. Validate whether `/STEP` needs a timer-generated **0.5–72 µs** low interval or whether a simpler response can provably satisfy both ROM and original-interface timing; prefer source-spec compliance.
2. Reconcile high-impedance/disable/power-save effects on the original **latched /TK0** with our continuously derived `track==0` model.
3. Confirm saturated-track boundary behaviour at track 79 and any beyond-track-zero handling needed for nonstandard software.
4. Validate **620 µs write-gate-to-step** exclusion against the eventual write-data input path and flash commit logic.
5. Determine whether any compatibility-critical driver requires an observable **/READY** not-ready phase after a step even if the selected data is immediately ready. Treat this as a measurement question, *not* a reason to add an arbitrary seek delay.
6. Establish timer/capture and RD-status publication latency under worst-case STM32 GCR generation and SPI flash activity.
7. Recheck any detailed differences for MFD-51W versus MFD-51W-03 using real hardware, while retaining the shared 20-pin profile with **pins 9 and 20 NC**.

**Related operations:** Section 8 specifies motor, ready and tachometer; Section 10 covers the read stream; [Section 11](06-part-3-media-architecture.md) consolidates shared media/timing decisions; future **Section 12** resolves seek/write exclusion and WR timing.

## 8. Operation: motor, ready handshake and tachometer

**Status:** Hardware speed zones, tachometer pulse count, status semantics, motor command mapping and actual Macintosh Plus ROM paths established. MCU timer allocation, undocumented stopped-tach level, power-save nuances and electrical validation remain open.

**Principal finding:** The 800K drive *self-regulates motor speed*, so the Macintosh Plus does **not** use PWM to control it; however, the original Plus `Sony_ReSeek` routine can **still measure TACH on an 800K drive** during periodic speed validation. A constant `/TACH` status is **not** a safe substitute for a timer-generated waveform.

### 8.1 Three distinct concepts: enabling the drive, commanding the spindle and being ready

The following are **not interchangeable**:

| Layer | Host/drive signal | Meaning |
|---|---|---|
| IWM drive-interface selection | IWM `mtrOn`/`mtrOff` access, producing `/ENBL.INT` (pin 14) | Connect/disconnect selected drive's control/status interface; **not** the Sony spindle command |
| Sony addressed motor command | CA1=1, CA0=0, SEL=0, CA2=0 or 1, strobed by LSTRB | Changes the drive's **/MOTORON** latch |
| Sony `/MOTORON` readback | RD (pin 16), project `0x4` / ROM `MtrOnAdr=8` | 0 = spindle **commanded on**; 1 = commanded off |
| Sony `/READY` | RD, project `0xD` / ROM `ReadyAdr=11` | 0 = disk/head/speed (or digital substitutes) ready for data transfer |
| Sony `/TACH` | RD, project `0x7` / ROM `TachAdr=14` | Time-varying motor tachometer signal; **60 pulses per revolution** |

**[DOC]** `/ENBL.INT=1` forces the original drive's `/MOTORON` latch **high (off)** and RD **high impedance**. A later interface re-enable does not automatically make the spindle on. **[ROM]** The Plus explicitly selects/enables the drive interface in `Sony_DiskSelect`, *then* reads `/MOTORON` and sends the motor command if necessary.

**[DECISION]** The STM32 therefore needs separate variables such as `host_enabled`, `motor_commanded`, `ready` and `tach_phase`, with distinct transition rules. It must not treat `/ENBL` low as “spindle running” or `/MOTORON=0` as automatic proof of readiness.

### 8.2 Motor control: command addresses, readback and shutdown

The command mapping is established in Section 6; repeated here to tie motor control to /READY and TACH:

| Operation | CA1 | CA0 | SEL | CA2 | Project address | ROM address | Effect |
|---|---:|---:|---:|---:|---|---|---|
| **MOTOR ON** (LSTRB rising) | 1 | 0 | 0 | 0 | `0x4` | `8` / `MtrOnAdr` | Assert `/MOTORON=0`; spindle commanded on |
| **MOTOR OFF** (LSTRB rising) | 1 | 0 | 0 | 1 | `0xC` | `9` / `MtrOffAdr` | Release `/MOTORON=1`; spindle commanded off |
| **Read /MOTORON** (no strobe) | 1 | 0 | 0 | 0 | `0x4` | `8` / `MtrOnAdr` | Selected RD level reports the latch |

**[DOC]** When `/ENBL` is high, `/MOTORON` is preset high. **[ROM]** [`Sony_PowerUp`](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1664-L1682) selects the drive, reads `/MOTORON`, returns without another command if it is already 0, or sends a **single LSTRB pulse at `MtrOnAdr`** otherwise.

**[ROM]** [`Sony_PowerDown`](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1638-L1655) can request immediate motor-off or schedule it using a VBL countdown. Immediate shutdown first strobes `MtrOffAdr` and then accesses IWM `mtrOff` to cut enable. [`Sony_DiskVBLTask`](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1991-L2024) handles deferred shutdown; the VBL task runs at roughly **0.5-second** intervals when appropriate.

**[DECISION]** On motor-off, stop advertising readiness and stop or invalidate the active read-data stream; preserve the virtual track number and mounted-media state. Do **not** flush/write-cancel unpredictably: outstanding buffered flash writes must have an independent safe-persistence policy (shared principles in Section 11; write details in Section 12).

### 8.3 Five speed zones: physical 800K mechanism versus electronic model

**[DOC]** The Sony 800K mechanism uses **constant-frequency GCR recording** over five zones of 16 tracks, changing the *physical spindle RPM* with track radius. Apple's 699-0452-A `2.16 (printed sheet 15) gives:

| Zone | Tracks | Original spindle speed | Time per revolution | Sectors/track/side | Target TACH pulse rate | Pulse period |
|---:|---|---:|---:|---:|---:|---:|
| 0 | 0–15 | **394 RPM** | ~152.284 ms | 12 | **394 Hz** | ~2538.071 µs |
| 1 | 16–31 | **429 RPM** | ~139.860 ms | 11 | **429 Hz** | ~2331.002 µs |
| 2 | 32–47 | **472 RPM** | ~127.119 ms | 10 | **472 Hz** | ~2118.644 µs |
| 3 | 48–63 | **525 RPM** | ~114.286 ms | 9 | **525 Hz** | ~1904.762 µs |
| 4 | 64–79 | **590 RPM** | ~101.695 ms | 8 | **590 Hz** | ~1694.915 µs |

Calculations: `revolution_time = 60 / RPM` seconds; `tach_pulses_per_revolution = 60`, so `tach_frequency = RPM` pulses/second and `tach_period = 1 / RPM` seconds.

**[DOC]** Section 2.16 allows **±2.5% speed variation while /READY is low**; the formatting timing figures (`3.4.8) use a tighter **±1.5%** criterion. The tachometer interface (`3.2.4.11, printed sheet 20) specifies **60 pulses/revolution** and **±0.2% period accuracy (STD)**. These refer to *different quantities*; do not interchange the motor's broad speed tolerance with a digital timer's period accuracy.

**[DECISION — logical track model]** No physical motor or per-zone GCR timing model is implemented. The STM32 uses **one global GCR bit-cell calibration** and logical **track descriptors** (sector count, valid bit length, data offset). The selected standard track's **sector count** determines the host-visible TACH frequency through a compatibility lookup, not through a motor simulation. GCR output stays at the same calibrated rate for every track. When changing tracks, update the stream and TACH coherently before asserting ready.

See [Part 2 / Section 10](06-part-2-serial-gcr-read.md) for bitstream details and [Part 3 / Section 11](06-part-3-media-architecture.md) for the agreed media/timing contract. Track bit length alone does **not** guarantee the frequency expected for TACH.

### 8.4 TACH is a selected waveform, not a fixed status bit

**[DOC]** The tachometer is selected on **RD** with:

~~~text
Project 0x7: CA2=0, CA1=1, CA0=1, SEL=1
ROM address 14: TachAdr
/ENBL.INT must be LOW to drive RD.
~~~

When `/TACH` is selected, RD changes between low and high in a periodic pulse waveform. The original specification gives the count and period accuracy, but **does not in the cited section define a universal high/low duty cycle or the motor-stopped idle level**. **[OPEN]** Those should be verified on actual hardware if tests establish a need for exact fidelity.

**[ROM]** The Plus speed-measurement code polls both levels, requiring **two successive reads low followed by two successive reads high** to accept an edge (see `SONY_WAITTACH`, lines 1948–1988). Thus **a constant level, a narrow pulse invisible to the 68000, or a waveform started only on the instant the host first selects TACH will not reproduce the required normal measurement reliably**.

**[DECISION]** Generate a stable periodic level while the logical spindle is operating; have the RD mux *observe* its current value when project `0x7` is selected. Do not reset the oscillator phase every time CA/SEL chooses TACH. **A 50% duty cycle is a provisional convenient choice**, with sufficiently long low/high intervals for the ROM's consecutive status reads; it is **not documented as a Sony duty-cycle requirement**.

**[DECISION]** Timer generation should continue independently of an occasional SPI flash operation, slow foreground task or GCR buffer preparation. The eventual STM32 implementation should prefer a hardware timer/output-compare or equivalently deterministic peripheral arrangement, not best-effort software toggling.

### 8.5 Critical ROM finding: 800K drives can still be tach-checked

The Macintosh Plus ROM uses TACH in **three related contexts**, and these must be distinguished.

**(a) 400K motor-speed calibration.** [`Sony_MakeSpdTbl`, lines 1778–1849](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1778-L1849) computes legacy PWM speed codes by measuring tachometer periods. **If `NewIntf=1` (800K), the routine skips that calibration**, taking an approximately **800-ms wait** to allow the disk to seat correctly. This is a *host-side* wait and not a reason to simulate motor inertia in the emulator.

**(b) Speed-zone setup.** [`Sony_SetSpeed`, lines 1250–1275](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1250-L1275) forces the legacy PWM speed **code to zero** when the drive is `NewIntf=1`. This does **not** mean the 800K mechanism runs at a fixed RPM. It regulates spindle speed internally, based on the radial zone. Our **pin 20 remains electrically disconnected**; the ROM's PWM buffer logic is not a speed-control connection to our MFD-51W emulator.

**(c) Speed validation during `ReSeek` — important exception.** [`Sony_ReSeek`, lines 832–882](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L832-L882) checks a per-zone `TSTblTicks` timestamp. When the zone has never been checked or the elapsed period exceeds `CkDelta` (normally **four minutes**), it branches to **`SONY_GETFREQ1` to measure TACH**. The test for `NewIntf` appears **later, on the adjustment branch**: an 800K drive cannot have its PWM code adjusted, but the *measurement still occurs*. This is an actual read-path risk if we omit TACH.

The ROM's [`DT_Sony_SpeedCheckTbl`, lines 1766–1776](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1766-L1776) contains target speed limits for each zone (approximately **±2%** around the nominal 394/429/472/525/590 RPM). If the measured speed is outside this range, `ReSeek` can retry and eventually report **`spdAdjErr = −79`**, including on an 800K drive for which it cannot adjust PWM.

**[ROM]** During formatting, [`SONY_SPDSEEK`, lines 2823–2849](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2823-L2849) **does skip** the tach-based format-speed measurement for `NewIntf=1`. The presence of that guard in the formatting path does **not** negate `ReSeek`'s different read-path logic.

**Consequence [DECISION]:** Accurate TACH is a **required** host-visible feature for our baseline Macintosh Plus ROM, not merely a legacy-400K compatibility option. Test initial access to each zone and repeated access after the four-minute recheck interval.

### 8.6 How the Plus ROM measures tachometer speed

**[ROM]** [`Sony_FigTrkSpd` and `SONY_WAITTACH`, lines 1879–1988](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1879-L1988):

1. Address **`TachAdr` (ROM 14 / project `0x7`)** and switch the IWM to status mode (`q6H`, read `q7L`).
2. Arm the **VIA timer 2**, clocked at approximately **783.36 kHz**, as a timeout.
3. Synchronise with a complete qualified **TACH low→high** transition.
4. Re-arm the timer and count **15 further TACH pulses** (one quarter of a 60-pulse revolution).
5. Determine the elapsed VIA counter ticks. The caller `SONY_GETFREQ1` takes **four such quarter-revolution measurements**, which together represent about one revolution.
6. Calculate the observed speed and compare it with the applicable zone limits.
7. When timer 2 expires without enough pulses, the routine returns **`noDriveErr = −64`** as a ROM-side failure, not a Sony-protocol error byte.

The key electrical lesson is that the original ROM **expects a real, regularly toggling status signal** during those measurements. A static `0` or `1` at project `0x7` cannot satisfy the synchronisation and counting loops.

### 8.7 /READY: definition, predicates and transitions

**[DOC]** Sony `3.2.4.12 (printed sheet 20) specifies **`/READY=0`** when a disk is present, the requested head position has settled, and the motor has reached the desired speed. Unlike `/MOTORON`, **/READY is a transfer-permission/status output**, not a simple motor-latch echo.

Read with **project `0xD` / ROM `ReadyAdr=11`**.

For the emulated drive, define:

~~~text
/READY = 0 only when:
    cartridge is logically present
    AND motor is logically commanded on
    AND current track/head GCR stream can be served correctly
    AND no conflicting write/media-change transition prevents transfer
    AND no relevant internal storage/stream fault is pending

otherwise /READY = 1
~~~

**[DECISION]** This is our **electronic readiness contract**; it replaces the original drive's mechanical head-settle and speed-stabilisation requirements. It is **not** a claim that the Sony mechanism has an SPI flash or buffer predicate.

**[DOC]** Sony `3.4.4, printed sheet 28, shows the motor-on/disk-in/enable timing:

| Original-drive event | Specified bound | Electronic interpretation |
|---|---|---|
| Motor-on to /READY assertion (low), with media and other conditions satisfied | **T1 ≤600 ms** | Physical spin-up ceiling, not a mandatory STM32 delay |
| Disk removed (`/CSTIN` rises) to /READY deassertion (high) | **T2 ≤0.5 µs** | **Fast status invalidation matters**; do not keep reporting ready for absent media |
| Disk inserted (`/CSTIN` falls) to /READY assertion (low) | **T3 ≤1.0 s** | Physical seating/spin-up ceiling, not a mandatory digital wait |
| Disable (`/ENBL` rises) to invalid/internal readiness condition | **T4 ≤50 ms to be valid**, as diagram labels it | RD is already **high impedance** when disabled; reset motor state promptly and verify what the source intends by T4 |

**[DOC]** The original format-speed timing (`3.4.8, printed sheets 33–34) allows **≤900 ms** from motor-on to settling within ±1.5% speed and **≤300 ms** from /READY assertion to that tighter speed criterion. Following a zone-changing step, the figure gives **≤200 ms** to the tight speed criterion and **≤50 ms** after /READY assertion. These are **physical-device maximums**; they are *not* instructions to block the STM32 for 900/200 ms.

**[DECISION]** Report ready as soon as the new track's correct timed data stream can be delivered. A preloaded track can become ready almost immediately. If SPI flash access or track preparation is pending, remain **not ready** until the data engine is genuinely prepared. Do not report `/READY=0` while pointing to a stale track, wrong speed-zone configuration, or an incomplete GCR buffer.

### 8.8 ROM motor power-up and READY polling

**[ROM]** [`Sony_PowerUp`, lines 1664–1682](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1664-L1682) works as follows:

1. Select/enable the drive interface through `Sony_DiskSelect`.
2. Read **`/MOTORON`** at ROM address 8 / project `0x4`.
3. If readback is already 0, **return without a new motor command**.
4. Otherwise, pulse LSTRB with `MtrOnAdr` (CA2=0).
5. For the **new-interface 800K path**, arrange an initial **75-ms** host wait and then enter `SONY_WAITREADY`. The legacy 400K path uses a different fixed-delay strategy.

**[ROM]** [`SONY_WAITREADY`, lines 1466–1485](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1466-L1485) then selects `ReadyAdr` and polls for **/READY=0**, at approximately **1-ms intervals** up to roughly **1 second**. **It continues even if the polling limit expires**; subsequent sector I/O can still fail if the drive is not actually ready.

The **75-ms motor-start wait** and the **800-ms new-interface seat wait** during `Sony_MakeSpdTbl` are **time already spent by the host** in their respective code paths. We should not double them by adding mechanical-delay simulation in the STM32.

**[ROM]** The post-seek use of `SONY_WAITREADY` differs in its initial delay (roughly **500 µs**; Section 7). Both paths read the **same** /READY sense bit; neither requires a separate ready pin.

### 8.9 Automatic insertion spin, disabling, ejection and idle TACH

The original 1986 specification (`3.6, `3.7, printed sheet 36) states that the mechanical drive may **automatically rotate the disk for up to two seconds on insertion** and has power-save exceptions during auto-rotation, ejection, head access and erase.

**Distinguish:**

- **Motor-control latch:** Host-commanded `/MOTORON`; reset to off by `/ENBL` high.
- **Physical spindle motion:** May briefly exist in the Sony mechanism during automatic seating even without a host motor command.
- **TACH output:** Reports physical speed when the motor is rotating; in our emulator it is digitally synthesized when needed.
- **/READY:** Remains a promise of valid transfer conditions, not simply evidence of spindle motion.

**[DECISION]** Our first emulator need not reproduce *automatic physical disk rotation* or spin-down inertia. Logical insertion and data preparation must be implemented, but do not schedule a two-second simulated “motor spin” merely because the original unit performs seating.

**[OPEN]** The manufacturer's cited /TACH description does **not** specify an exact static idle level with motor stopped, nor exactly which insertion-auto-spin states may be visible through its selected RD multiplexor. Use **“no pulses while virtual spindle stopped; idle level provisionally high”** as a testable implementation hypothesis, **not a verified Sony electrical behaviour**.

**[DOC]** The original drive is designed to finish an already-triggered **eject** even after /ENBL goes high. That does not mean the motor-control latch should remain on. Eject logical state and media-present transitions belong in Section 9.

### 8.10 Proposed STM32 timing/ready architecture

**[DECISION]** Use logical track/side metadata and a **single global GCR cell timing reference**. The TACH timer selects its host-compatible frequency from the standard track's **sector count** (Part 3 / Section 11), independent of the data-cell clock:

~~~text
State:
    host_enabled                  # /ENBL.INT pin 14; interface gating
    motor_commanded               # /MOTORON latch
    track                          # 0..79, from Section 7
    selected_side
    sector_count                   # selected logical track metadata
    valid_track_bits               # explicit cyclic bit length
    global_bit_cell_period         # one stored calibration for cartridge
    tach_period                    # lookup(sector_count), NOT per-zone GCR timing
    tach_phase                     # independent oscillator phase
    media_present
    track_stream_ready
    media_transition_or_fault
    write_transition_active

On qualified MOTOR-ON command (Section 6):
    motor_commanded = true
    enable_virtual_tach_if_applicable()
    make_current_track_stream_active_when_prepared()

On qualified MOTOR-OFF command or /ENBL disabled:
    motor_commanded = false
    stop_virtual_tach()
    invalidate_data_transfer_readiness()
    preserve_track_and_media_state()

On accepted STEP (Section 7):
    invalidate stale stream
    load new track/side metadata + raw bitstream
    install TACH frequency from sector_count
    keep the same global GCR bit-cell period
    /READY = 1 until correct stream is available

Compute ready:
    ready = motor_commanded AND media_present
            AND track_stream_ready AND NOT media_transition_or_fault
            AND NOT write_transition_active_if_it_blocks_transfers

RD multiplexer when /ENBL is LOW:
    if project selector == 0x4: RD = motor_commanded ? 0 : 1
    if project selector == 0x7: RD = current_tach_phase_level
    if project selector == 0xD: RD = ready ? 0 : 1
    # other selectors per Section 4

RD while /ENBL is HIGH: HIGH-IMPEDANCE.
~~~

**Implementation notes:**

- Prefer an **STM32 hardware timer** (or equally deterministic hardware output) for `/TACH`; choose prescaler/ARR values whose rounding error remains comfortably within the desired tach accuracy across all five rates.
- **Keep tach phase independent of which status is currently selected.** Selecting TACH should reveal the oscillator state, not restart it or synthesize an artificial pulse.
- On a track change, install the sector-count-derived TACH frequency at a controlled timer boundary, load the new bitstream length and retain the same global GCR bit-cell rate. Maintain reasonable coherence between the TACH status and track duration before signalling ready.
- RD is a **shared high-speed physical output**. The selector and enable circuitry must arbitrate correctly between static status, TACH toggles and GCR transitions, with the `≤0.5 µs` status-change/output-disable figures to be validated in Section 15.
- Keep /READY derivation synchronized with any flash/media buffering operation, but do not block low-latency status or TACH generation on SPI flash commands.
- An **MCU clock/power brownout** must tri-state RD safely and must not cause the Macintosh to see a falsely ready or phantom-motor-on drive.

This is the **functional architecture**. No STM32 part number, clock tree, DMA channel, GPIO pin assignment or hardware flip-flop/buffer scheme is approved yet.

### 8.11 Verification matrix and unresolved details

| Test or observation | Expected result | Evidence |
|---|---|---|
| Drive disabled (`/ENBL=1`) | RD high impedance; `/MOTORON` resets off | [DOC] |
| Enable drive but do not issue MOTOR ON | Motor-status readback off; /READY not asserted merely by enable | [DOC]/[DECISION] |
| Strobe MOTOR ON (`0x4`) | `/MOTORON=0`; virtual tach starts; /READY asserts only when real data conditions hold | [DOC]/[DECISION] |
| Strobe MOTOR OFF (`0xC`) | `/MOTORON=1`; transfer readiness ends | [DOC]/[DECISION] |
| No cartridge but motor commanded on | `/CSTIN=1`, `/READY=1`; exact no-media TACH behaviour [OPEN] | [DOC]/[OPEN] |
| Valid cartridge, motor on, stream ready | `/READY=0`, matching the ROM's status sense | [DOC]/[ROM] |
| Cartridge removed while ready | `/READY` quickly becomes 1; no stale bitstream is offered | [DOC]/[DECISION] |
| TACH selected at track 0 | Period ~2538.071 µs (394 Hz); alternating sense samples | [DOC]/[ROM] |
| TACH selected at track 79 | Period ~1694.915 µs (590 Hz); alternating sense samples | [DOC]/[ROM] |
| Cross each of four zone boundaries | TACH frequency changes to the new zone's exact nominal rate | [DOC]/[DECISION] |
| Read while `ReSeek` checks a zone initially or after ~4 minutes | VIA timer sees 15 edges per quarter revolution; ROM speed check succeeds | [ROM] |
| 800K `MakeSpdTbl` called | ROM takes its 800-ms seating wait; no 400K PWM calibration demanded | [ROM] |
| 800K `SpdSeek` during format | ROM skips legacy format speed calibration; data-path timing still correct | [ROM] |
| Stop/restart motor; switch TACH selector repeatedly | No fake pulses due solely to selector switching; no mechanical wait | [DECISION] |
| SPI flash busy while motor logically running | Hardware-timed TACH unaffected; READY may deassert if stream unavailable | [DECISION] |
| Invalid idle TACH or missing pulses (negative test) | ROM can timeout with `noDriveErr`; subsequent read error recovery may follow | [ROM] |

**Open questions:**

1. Determine original **/TACH stopped-state level**, pulse duty and whether an auto-insertion spin can be observed at RD in the common MFD-51W/-03 internal configuration.
2. Measure full **motor-off-to-/READY-high** electrical latency and interpretation of `3.4.4 T4, particularly when /ENBL is disabling RD at the same time.
3. Determine how GCR streaming resumes on power-up and after a zone change without breaking the ROM's 15-pulse/quarter-revolution speed measurements.
4. Verify whether the Plus or third-party software tests the *temporal* sequence of motor-on and /READY even when the virtual track is immediately prepared.
5. Validate the timer's effective edge-period error/jitter under full SPI flash activity, GCR bitstream generation, selector changes and interrupt load.
6. Resolve exact behaviour for **read during disk insertion**, **motor-on with no cartridge**, **eject in progress**, and flash commit during power-down in Sections 9, 12 and 14.
7. Reconcile readiness and tach performance against a real MFD-51W or MFD-51W-03 once hardware is available; preserve **pins 9 and 20 NC** throughout.

## 9. Operation: disk insertion, protection and eject

**Status:** Apple 1985/1986 hardware sense semantics, Macintosh Plus ROM media discovery/first-access/eject sequence and eject timing investigated. Cartridge electrical/mechanical implementation, flash write-persistence guarantees and unusual media-fault responses are proposed, **not yet validated**.

**Core principle:** The Macintosh sees an **electrical/logical removable disk**, not the SPI flash contacts or physical release mechanism. Keep the following distinct: (1) physical cartridge present, (2) cartridge electrically qualified, (3) media logically inserted/exposed through `/CSTIN`, (4) data stream ready through `/READY`, (5) write permission through `/WRTPRT`, and (6) a cartridge that has been logically ejected but has not yet physically left the mechanism.

### 9.1 Interface signals, selectors and software responsibilities

There are **no separate disk-present, write-protect, or eject-complete pins** on the common internal MFD-51W host connector. These are drive statuses selected onto RD (pin 16). Commands use the same selection signals and an LSTRB rising edge, as described in Sections 4 and 6.

| Operation | Interface | Project selector | ROM address | Meaning |
|---|---|---|---|---|
| Read disk-in-position status | RD via CA0/CA1/CA2/SEL | `0x1` | `2 / DIPAdr` | `/CSTIN=0`: disk-in position; normally inserted media |
| Read write-protect status | RD via CA0/CA1/CA2/SEL | `0x3` | `6 / WrProtAdr` | `/WRTPRT=0`: protected **or no disk** |
| Read transfer readiness | RD via CA0/CA1/CA2/SEL | `0xD` | `11 / ReadyAdr` | `/READY=0`: data transfers may proceed |
| Read drive installed | RD via CA0/CA1/CA2/SEL | `0xE` | `13 / DrvExstAdr` | `/DRVIN=0`: mechanism/emulator installed, even if empty |
| Read eject latch/operation | RD via CA0/CA1/CA2/SEL | `0x6` | `12 / EjectLAdr` | EJECT readback exists in Sony table; exact original-Plus usage remains **[OPEN]** |
| **Command software eject** | LSTRB rising with CA1=1, CA0=1, SEL=0, CA2=1 | `0xE` | `13 / EjectHAdr` | Begin eject operation; **not pin 9** |
| Host drive selection | `/ENBL.INT`, pin 14 | — | IWM `intDrive` + `mtrOn` | Must be active for status access and command |
| Write-data acceptance | `/WRREQ`, WR, enable and media protection | — | IWM write path | Media principles Section 11; detailed write path Section 12 |

**[DECISION]** Emulator connector **pins 9 and 20 are NC**. The revision-specific hardware eject input on certain Sony drives is not used. **Software eject through the regular addressed command is mandatory**.

### 9.2 Exact sense polarities and the empty-drive case

**[DOC]** Apple 699-0321 (1985), `3.2.4.8–3.2.4.9, and 699-0452-A (1986), same subsections, specify:

- `/CSTIN` reads **0** if a diskette is inserted **OR** if the original eject/inject mechanism is at the disk-in position **even without a diskette**. The latter is a mechanical exception, not evidence of valid magnetic data.
- `/WRTPRT` reads **0** when a **write-protected disk is inserted OR no diskette is inserted**. It is **1 only with inserted writable media**.
- `/READY=0` additionally requires appropriate head/motor/readiness conditions. Disk presence by itself is not readiness.
- `/DRVIN=0` identifies a connected mechanism independently of media insertion.

**[DECISION]** Because our emulator has no Sony mechanical injected-without-disk position, the initial host-visible model maps `/CSTIN` to **logical media presentation** (not merely an unqualified contact switch). There is no reason to invent Sony's empty-mechanism exception unless testing reveals a dependency.

| Emulator state (internal interface enabled) | `/CSTIN` | `/WRTPRT` | `/READY` | `/DRVIN` | Interpretation |
|---|---:|---:|---:|---:|---|
| **Empty / logically ejected** | **1** | **0** | **1** | **0** | Installed drive, **no disk**, not ready, protected by default |
| Insertion detected, contacts/image not yet qualified | **1** | **0** | **1** | **0** | Do not announce unstable/unusable media |
| **Inserted, writable, ready** | **0** | **1** | **0** | **0** | Normal writable disk |
| **Inserted, write-protected, ready** | **0** | **0** | **0** | **0** | Reading permitted; writing denied |
| Inserted, writable, **not ready** | **0** | **1** | **1** | **0** | Correct media presence; preparation/fault keeps transfer disabled |
| Inserted, read-only, **not ready** | **0** | **0** | **1** | **0** | Present but not transfer-ready |
| Eject command accepted; media not yet logically withdrawn | **0** | As above | **1** | **0** | Transient; **no new writes** may begin |
| Ejected and awaiting physical removal | **1** | **0** | **1** | **0** | **Must not reannounce** the still-connected cartridge |

This is a **proposed emulated status contract**, distinct from the physical Sony edge case. A **blank/unformatted but electrically healthy** cartridge may be logically inserted and ready for formatting: do not mistakenly equate “no recognized Macintosh filesystem” with “no disk.”

**Important correction to oversimplified status logic:** Computing `/WRTPRT = !write_protected` unconditionally is **wrong**. A suitable derived expression is:

~~~text
/WRTPRT = 1 only if (logical_media_present AND media_write_permitted)
           otherwise 0
~~~

The lack of a slash prefix on the project's boolean variable `write_protected` is intentional.

### 9.3 Macintosh Plus periodic disk insertion detection

**[ROM]** [`Sony_DiskVBLTask`, lines 1991–2024](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1991-L2024) is a periodic task, normally invoked at **30 VBL ticks (~0.5 s)**. It skips media inspection if a driver request is active and may first attend to pending motor power-down.

For floppy drives, [`Sony_Check4Disk` / `SONY_CKDIP_SENSE`, lines 2028–2070](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2028-L2070) works approximately as follows:

1. Confirm the drive is installed in the ROM's drive table.
2. Inspect the ROM's **software** `DiskInPlace` state. **1 or 2 means already recognized** and normally avoids another insertion probe.
3. If it is **0**, select the drive and read `DIPAdr`, the Sony `/CSTIN` status.
4. When `/CSTIN=0` (disk in), post **`diskInsertEvt` (event 7)** with the drive number.
5. On successful event post, set software `DiskInPlace=1` (detected, but not yet prepared), and invalidate the driver's software track position.
6. Further handling occurs on first disk access (next subsection).

The **STM32 does not post Macintosh events**, inspect System files or manipulate the ROM's `DiskInPlace` variable. It only presents the electrical `/CSTIN` status. The Macintosh's existing ROM/OS decides when to announce and mount the inserted disk.

**[ROM]** After software eject the driver loads `DiskInPlace=EjectWait`, normally `−4`, and increments it on successive VBL calls until it reaches zero before allowing normal insertion detection again. At nominal 0.5-s cadence this is roughly a **two-second *host-side*** suppression interval. **[DECISION]** Do not add a second arbitrary two-second “mechanical eject delay” in the STM32; instead ensure that the same ejected cartridge cannot be rediscovered through `/CSTIN` once the ROM's suppression expires.

### 9.4 First disk access: presence, recalibration and write protection

**[ROM]** [`Sony_DiskPrime`, lines 545–583](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L545-L583):

1. Verify the drive is installed, select it, and **read `/CSTIN` again**, even if the ROM previously believed a disk was inserted. This handles a disk that disappeared outside the normal eject path.
2. If `/CSTIN=1`, clear the ROM's `DiskInPlace` and return **`offLinErr = −65`**. In this path the ROM also disables the drive.
3. If present but not previously prepared (software `DiskInPlace≠2`), perform a first-access sequence: **`Sony_FastPower`**, **recalibration to track 0**, **`Sony_MakeSpdTbl`**, then **read `/WRTPRT`**.
4. Cache the protection result in the ROM's `WriteProt` variable, set software `DiskInPlace=2` (prepared), and proceed.
5. On a subsequent *write* request with cached write protection active, return **`wPrErr = −44`**. Read requests are still permitted.

**[ROM]** For an 800K/new-interface drive, `Sony_MakeSpdTbl` takes an approximately **800-ms host-side disk-seating wait** rather than performing 400K PWM speed calibration ([lines 1778–1795](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L1778-L1795)). Do not add a matching delay for mechanical realism in the STM32.

**[DECISION]** A cartridge's write-protect setting should be stable while it is logically mounted, or its change must be handled conservatively (e.g., force read-only and prohibit further writes) because the ROM **caches** the original readback. The emulator itself must enforce write protection **even if the host's cached value becomes stale**.

**Important:** Media being *formatted/unformatted* is not a valid reason to deassert `/CSTIN`. An inserted blank disk still needs to be visible for host formatting, even if ordinary sector reads fail until formatting.

### 9.5 Software eject: command and required host-visible sequence

**[DOC]** Original Sony 699-0452-A, `3.2.4.4, `3.4.7 (printed sheet 32) and `2.17.1 (sheet 15) define a **software eject command**. On a qualified **LSTRB rising edge** with `CA1=1, CA0=1, SEL=0, CA2=1` (project `0xE` / ROM `EjectHAdr=13`), the drive begins eject. The **EJECT control state goes high** and returns low upon the rising edge of `/CSTIN` (disk exits) or after **2 seconds maximum**. The mechanism's specified eject completion time is **≤1.5 seconds**.

**[ROM]** [`SONY_CTL_EJECT` / `SONY_NEWDRVEJECT`, lines 440–480](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L440-L480) performs:

1. Invalidate the Macintosh's track cache and command the spindle **off**, without immediately leaving the interface disabled.
2. Select/keep the drive enabled, then **wait 200 ms** before issuing eject.
3. Address **`EjectHAdr` (13)**; raise LSTRB to initiate eject and **lower it promptly** for a newer 800K drive. The **older 400K** path instead uses a much longer ~0.75-s strobe; it is **not** the MFD-51W requirement.
4. For the 800K path, poll **`/CSTIN`**, first after **100 ms** and then roughly every **100 ms**, with **up to 16 checks (~1.6 s)**. Success is `/CSTIN=1`.
5. Wait an additional **1 ms** after completion or the bounded poll loop, then set the ROM's negative `EjectWait` suppression counter. **The routine follows its success-completion path even after polling expires**; it is not a reliable physical-cartridge-release acknowledgement.

| Sony mechanism timing parameter | Original electrical/mechanical requirement | STM32 relevance |
|---|---|---|
| Motor off before LSTRB eject | **≥200 ms** | Host does this; no extra simulated delay required |
| Eject LSTRB pulse | **≥1.0 µs; ≤300 ms** | Accept rising edge; do not require a held 400K-style pulse |
| Eject completion (`/CSTIN` rises) | **≤1.5 s** | Bound logical removal/acknowledgement when eject succeeds; flash safety still mandatory |
| Keep `/ENBL` low after `/CSTIN` rises | **≥150 µs** | The Plus driver waits at least another 1 ms |
| Internal EJECT high duration | **≤2 s** | Ensure eject state eventually returns inactive; distinguish from cartridge mechanical travel |

**[DOC]** An already-initiated eject operation **may finish after /ENBL becomes high**. RD is nevertheless **high impedance while /ENBL is high**; internal eject/cartridge state is a separate matter.

### 9.6 Logical ejection versus physical cartridge release — a necessary new state

Our selected media concept is a **3D-printed 3.5-inch floppy-shaped cartridge containing SPI flash and mating contacts** ([Document 05](05-cartridge-contact-concepts.md)). Its **actual insertion, retention and release mechanisms have not been designed or approved** ([Document 04](04-chassis-and-mechanics-quickstart.md)). Do not assume the cartridge is physically expelled, loses SPI contact, or has an eject motor identical to Sony's.

A robust design distinguishes at least these events:

1. **Eject requested:** accept the LSTRB command. Stop accepting new writes for the current medium; prevent further stale GCR data. Mark the operation as pending.
2. **Safe-to-withdraw:** any *already accepted* writes are committed safely or their recovery is resolved. This is not the same instant as receiving the command.
3. **Logical eject acknowledged:** report `/CSTIN=1`, `/WRTPRT=0`, `/READY=1`; invalidate the mounted image/track cache. This is the signal the Macintosh ROM actually waits for.
4. **Physical release requested/completed:** drive any future latch/actuator if one is selected; handle sensor/contact changes safely. This is an *independent mechanical subsystem*, not a prerequisite for the interface design to be studied.
5. **Awaiting full removal:** **even if the cartridge remains seated and contacts still conduct**, keep it **logically absent** until a confirmed, debounced removal-and-new-insertion cycle (or another explicitly designed user action) occurs.

**[DECISION]** Introduce **`EJECTED_AWAIT_REMOVAL`** (or equivalent) so the same cartridge does **not** reappear as a fresh disk after the ROM's own ~2-second `EjectWait` suppression expires. Merely mapping `/CSTIN` directly to contact continuity would cause the Macintosh to rediscover an already-ejected image.

**[OPEN]** The ordering of actual physical release versus completion of flash commits depends on the final retention and actuator design. The electronics must not permit the cartridge to be withdrawn while the design still depends on it to persist data **without an explicit, tested failure-recovery guarantee**. A future paperclip/manual-release path may defeat an electronic interlock and needs separate protection.

### 9.7 Insertion qualification and cartridge identity

The 3.5-inch form factor is a **physical package**, not a host-visible indication of the SPI flash protocol. Candidate cartridge-side signals already discussed in [Document 05](05-cartridge-contact-concepts.md) include presence detection, cartridge identity/profile and an optional write-protect input.

**[PROPOSED]** A safe insertion sequence is:

1. Detect that a physical cartridge is being inserted, and **debounce/validate its settled position** with a yet-to-be-selected sensor/contact arrangement.
2. Establish safe power/contact continuity (including ground-first if designed), start/identify SPI flash and ensure that partial contact cannot write to flash or spuriously reset/drive the host lines.
3. Validate that the cartridge has usable backing storage (including a valid *blank* medium representation), learn its geometry/capacity and write policy, and prepare enough data/state to answer future reads correctly.
4. **Only after qualification**, set logical media present: `/CSTIN=0`. The ROM can now discover the disk on its next polling cycle.
5. Assert `/READY=0` only when separately allowed by motor state and data-engine readiness. Insertion alone does not imply ready.

Distinguish **a blank, format-capable medium** from **unreadable/failed flash**:

- A *blank but healthy* cartridge should remain logically present and be format-capable under whatever media-layout scheme we choose.
- A *present but temporarily not-ready* cartridge reports `/CSTIN=0`, `/READY=1`. This avoids pretending an electrical storage fault is an eject event.
- A *failed/unqualified* cartridge should not be advertised as a newly readable disk. Whether to keep it logically absent or present-but-not-ready after a fault is an **[OPEN]** error-policy choice that must be made deterministically.
- A *write-protected* cartridge is still a valid readable disk; its `/WRTPRT=0` response is not a failure.

**[DECISION]** Standard structured 32+703-byte encoded-GCR records (including preserved address/data headers), optional exact RAW_TRACK representation, >=3 complete >=12-KiB buffers, global GCR calibration and extensible track metadata are defined in [Part 3 / Section 11](06-part-3-media-architecture.md). The precise on-flash binary ABI, atomic flash operations and failure recovery remain **[OPEN]**.

### 9.8 Write-protect sources, caching and enforcement

Possible causes of a non-writable cartridge include a deliberate write-protect setting, a read-only media profile, a hardware write-protect contact/switch (if selected), or protective firmware mode after detecting a storage fault.

**[DECISION]** The host-visible `/WRTPRT` is derived from **logical presence AND permission to modify the current cartridge**. It must return **0** for **no media** and for **inserted protected media**, and **1** only when the current logical disk is present and writable.

**[ROM]** The Plus driver samples `/WRTPRT` on the first access and stores a cached value (`WriteProt`). It is not necessarily re-read before every sector write. Therefore:

- STM32 firmware must **enforce write protection independently** of the host's cached bit.
- A change from writable to protected while mounted may result in the host attempting writes it previously expected to succeed; do not modify flash. Define a controlled media-change/error policy (not invented successful writes).
- A change from protected to writable should not silently override an OS-cached read-only decision. A **logical eject and fresh insertion** is the conservative way to reannounce a changed policy.
- Some write-protect transitions may be implemented purely in cartridge firmware metadata and may not correspond to a mechanical floppy tab. That is permitted provided host behaviour is correct.

**[DOC]** Apple `3.2.5 specifies recording only when **/WRTGATE=0**, **/ENBL=0** and media **not write-protected**. Section 12 must specify how the Macintosh Plus's `/WRREQ` maps to Sony's gate and how invalid writes are rejected/ignored without ever corrupting flash.

### 9.9 Abrupt removal, power interruption and safe flash writes

Unlike magnetic media, an SPI flash cartridge may be vulnerable to power/contact interruptions during an erase/program operation. A physically pulled cartridge might become electrically absent **before** firmware has finished persistence.

**[DESIGN REQUIREMENT / OPEN IMPLEMENTATION]** Establish clear durability semantics before promising a functional eject:

- **Completed writes must not be silently lost** when the Macintosh receives a successful logical eject. Any buffering or flash journal must be committed/recoverable beforehand.
- Prevent **new writes** once software eject has been accepted for that cartridge; settle or reject in-flight transfers according to the future **Section 12** write-operation contract (shared persistence principles in Section 11).
- Never continue issuing SPI flash erase/program commands to contacts known to be open or unstable.
- On unexpected withdrawal: **immediately mark media not ready**, remove `/CSTIN` after qualified removal, stop disk-stream access, and protect later cartridge insertions from stale buffers belonging to the earlier cartridge.
- Power-fail detection, emergency energy storage, atomic page/sector mapping, verification, journaling and media generation numbers are possible approaches, **not approved designs**.
- Do not make a false **successful eject** claim to the host while outstanding data could be lost. If the durability operation cannot complete within the Sony/ROM eject window, treat it as a **design problem to solve and test**, not permission to violate data integrity.
- If power loss or manual removal occurs despite safeguards, restore from a consistent committed state after reinsertion where possible, and surface failures through appropriate subsequent I/O behaviour. The Sony drive interface has **no extra generic SPI-flash error-status register**.

**[OPEN]** Choose physical contacts, sequencing, removal interlock and flash write algorithm after power and worst-case programming latency are measured. The present discussion defines the correctness goal, not a guarantee that an arbitrary flash device or detachable connector can meet it.

### 9.10 Preliminary media/eject state machine and outputs

**[PROPOSED]** Separate physical-contact detection from host-visible media state. This is functional pseudocode only; the actual STM32 event scheduling, timer qualification and storage handling remain open.

~~~text
State:
    EMPTY                       # no logical media, no recently ejected cartridge
    QUALIFYING                  # contacts/media identification in progress
    PRESENT_UNREADY             # accepted media but GCR stream not yet ready
    PRESENT_READY               # accepted media; transfer-ready if motor on
    EJECT_PENDING               # host requested eject; block new writes
    EJECTED_AWAIT_REMOVAL       # logically absent, even if contacts still present
    MEDIA_FAULT                 # exact external policy to be determined

Physical inputs (internal, not Sony pins):
    cartridge_sensor
    contact_valid
    cartridge_identity
    write_protect_source       # optional physical pin/metadata
    media_storage_health

On STM32 startup:
    host_RD = HIGH_IMPEDANCE until safe to drive
    motor_commanded = OFF
    logical_media_present = false until qualified
    determine physical cartridge/contact state
    if reliable cartridge already inserted:
        qualify and present it (policy for boot-time cartridge [OPEN])

On debounced *new physical insertion* from a genuinely absent condition:
    state = QUALIFYING
    stop disk data, prohibit writes
    if cartridge identification and backing media qualify:
        state = PRESENT_UNREADY
        logical_media_present = true       # /CSTIN becomes 0
        write_permission = mounted_media_policy
        prepare selected data stream
        if data prepared:
            state = PRESENT_READY
    else:
        present no false READY; resolve fault policy

On motor/track status changes:
    derive /READY from logical_media_present, motor, stream validity,
                        and absence of conflicting transition/fault

On LSTRB software-EJECT command, if a logical disk is present:
    state = EJECT_PENDING
    block new writes; stop exposing stale data
    complete or recover all accepted write operations safely
    if durable-and-safe:
        unmount logical disk; /CSTIN=1; /WRTPRT=0; /READY=1
        state = EJECTED_AWAIT_REMOVAL
        request physical release if mechanism selected
    else:
        retain safe fault/pending state; do NOT claim successful eject
        # timeout/error policy [OPEN]

On cartridge physically removed:
    stop GCR and storage access; protect open contacts
    cancel invalid in-memory references to removed medium
    /CSTIN=1; /WRTPRT=0; /READY=1
    if state == EJECTED_AWAIT_REMOVAL:
        state = EMPTY
    else:
        state = EMPTY                        # unexpected/manual removal
    # any incomplete flash writes require recovery/error handling

When state == EJECTED_AWAIT_REMOVAL:
    always present logical no-disk, regardless of remaining contact continuity
    do not automatically re-present the same cartridge after ROM EjectWait

Derived RD, when enabled:
    project 0x1 /CSTIN  = logical_media_present ? 0 : 1
    project 0x3 /WRTPRT = (logical_media_present && write_permission) ? 1 : 0
    project 0xD /READY  = can_actually_transfer_data() ? 0 : 1
    project 0xE /DRVIN  = 0         # emulator itself remains installed
    project 0x6 EJECT   = eject_command_state  # readback semantics pending

When /ENBL disabled:
    RD = HIGH_IMPEDANCE
    # do not erase cartridge/eject state, and do not abort safe persistence
~~~

The **`PRESENT_READY` name is not sufficient by itself** to assert `/READY=0`: the logical motor command and selected stream must also allow transfer. This state model is intentionally conservative.

**Implementation caution:** The simulated drive must not manufacture a media eject merely because the Macintosh deselects the drive or commands MOTOR OFF. **Eject is a separate command** and physical cartridge removal is another event.

### 9.11 Verification matrix and open decisions

| Test case | Expected electrical/logical result | Confidence |
|---|---|---|
| Power on, no cartridge | `/CSTIN=1`, `/WRTPRT=0`, `/READY=1`; `/DRVIN=0` | [DOC]/[DECISION] |
| Empty cartridge slot, no software eject | Remains empty, no disk-in event from ROM | [ROM]/[DECISION] |
| Fully qualified, writable cartridge inserted | `/CSTIN=0`, `/WRTPRT=1`; ready subject to motor/stream | [DOC]/[DECISION] |
| Fully qualified, write-protected cartridge inserted | `/CSTIN=0`, `/WRTPRT=0`; reads allowed, writes denied | [DOC]/[ROM] |
| Insert blank but format-capable cartridge | Host sees disk present even if filesystem/sector fields are absent | [DECISION] |
| Cartridge contact bounce during insertion | No repeated present/absent events; no flash write during partial contact | [PROPOSED] |
| ROM VBL poll detects insertion | Plus posts `diskInsertEvt=7`; software `DiskInPlace` goes 0→1 | [ROM] |
| First read/write after insertion | ROM checks `/CSTIN`, motor, recalibration, `/WRTPRT`; caches prepared state | [ROM] |
| Read after unexpected manual removal | ROM sees `/CSTIN=1` and returns `offLinErr=-65` | [ROM] |
| Host attempts write to protected media | Host may return `wPrErr=-44`; firmware also blocks underlying flash modification | [ROM]/[DECISION] |
| Software eject with no pending writes | Qualified command causes `/CSTIN=1` within the compatible window; /READY=1 | [DOC]/[DECISION] |
| Software eject while flash programming | No successful logical eject before accepted data are durable/recoverable | [DESIGN REQUIREMENT] |
| Ejected cartridge remains physically seated | `/CSTIN` stays **1**, preventing rediscovery after host suppression expires | [DECISION] |
| Cartridge removed fully, later reinserted | A **new** qualified insertion can be announced; no stale track/write buffers | [DECISION] |
| /ENBL high during accepted eject | RD tri-state; internal safe eject progression not cancelled just by deselection | [DOC] |
| Software eject times out | Do not imply ROM `noErr` proves physical release; investigate error and release policy | [ROM]/[OPEN] |
| Media removed while data stream active | Stop stale stream, report not-ready and absent; guard flash contacts | [DECISION] |
| Write-protect changed without removing media | Firmware never writes when protected, regardless of cached host status | [DECISION] |
| Power fails during dirty flash commit | No silent cross-cartridge corruption; recover/indicate fault with selected scheme | [OPEN] |

**Open items to settle before hardware/firmware approval:**

1. The detailed **physical cartridge insertion/retention/eject actuator** and how a paperclip/manual release interacts with SPI flash safety.
2. Precise strategy for **durable write acknowledgement** and maximum flash commit/erase duration relative to the 1.5-second Sony eject completion requirement.
3. Whether a media fault before logical insertion should be **logically absent** or **present but not ready**, and when to promote a temporarily unready medium back to ready.
4. Whether the chosen cartridge design includes a **physical write-protect sensor**, flash metadata flag, both, or another scheme; ensure host protection remains stable during a mounted session.
5. Decide the explicit **EJECT state readback** at project `0x6` after testing original MFD-51W output and ROM/third-party use.
6. Determine acceptable detection/debounce and RD-response timing for **unexpected removal** and re-insertion; keep flash electrical safety separate from host-visible timing.
7. Validate both **Macintosh Plus ROM insertion/eject paths** and nonstandard/third-party software behaviour with a real host or instrumented emulator.
8. Verify that the physical design's *partial-eject position* does not accidentally requalify the same cartridge while still electrically connected.
9. Keep software eject on the common LSTRB command interface with **pins 9 and 20 left unconnected**, independent of the chosen cable revision.

**Related sections:** Section 10 (GCR reading and blank/unformatted media), [Section 11](06-part-3-media-architecture.md) (raw-GCR media, calibration, logical track metadata), [Section 12](06-part-4-serial-gcr-write.md) (write gate, capture and persistence requirements), Section 14 (power-fail/media faults), Section 15 (electrical response timing), and Section 16 (STM32 state and peripherals).

## Continuation and future parts

This continuous protocol is split by responsibility. **Part 3 / Section 11** sits between **Part 2 / Section 10** reading and the now-developed **[Part 4 / Section 12](06-part-4-serial-gcr-write.md)** writing procedure. Earlier planned Sections 12–17 were shifted to **13–18**.

- **[Section 10 — Serial GCR reading (Part 2)](06-part-2-serial-gcr-read.md#10-operation-serial-gcr-disk-data-reading):** detailed read path and freshly formatted stream.
- **[Section 11 — Shared media, calibration and cartridge architecture (Part 3)](06-part-3-media-architecture.md#11-shared-structured-gcrraw-track-media-timing-and-cartridge-architecture):** current, agreed principles and open implementation questions.
- **[Section 12 — Serial GCR writing (Part 4)](06-part-4-serial-gcr-write.md#12-operation-serial-gcr-writing):** developed, with hardware splice/flash persistence open.
- **Sections 13–17:** additional protocol, electrical, implementation and verification work.
- **Section 18:** future consolidated decision log; historical notes retained in the appendix.

### Preserved research scaffolds for planned Sections 12–17

### Section 12: Operation: serial GCR writing — [developed in Part 4](06-part-4-serial-gcr-write.md#12-operation-serial-gcr-writing)

Sony gate and both-edge WR capture, IWM write handshake, ROM sector writes, full-track format, SYNC6 calibration, RAM buffer design and persistence engineering gates are developed in [Part 4](06-part-4-serial-gcr-write.md). This is a navigation stub rather than a duplicate specification.

### Planned Section 13: Operation: format, sector geometry, sides and 400K compatibility

**TO DEVELOP.** ROM `Sony_CtlFormat`, `SONY_DOFORMAT`, `SONY_FILLINMARKS`. Standard formatted track sector counts 12/11/10/9/8 across 80 radial tracks; head selection, interleave, checksums, format bytes, 400K compatibility, standard-track generation and fit/seam constraints. The STM32 uses **logical sector counts/bit lengths**, not a physical spindle model.

### Planned Section 14: Operation: error handling and unusual sequences

**TO DEVELOP.** No disk, bad data, repeated step, out-of-range track, not-ready timeout, eject/write collision, unexpected removal, underrun, power loss and recovery.

### Planned Section 15: Electrical requirements and timing

**TO DEVELOP.** Pin voltage thresholds, RD tri-state/output buffer, host load, command and status response limits, GCR transition width/spacing, 5 V ↔ STM32 interfacing, power and pin 9/20 isolation. Link to [Document 02](02-macintosh-20-pin-interface.md).

### Planned Section 16: STM32 implementation architecture

**TO DEVELOP.** Selector logic, LSTRB capture, RD mux, GCR timer/DMA, WR transition capture, shared track position, global calibration, removable SPI flash cache/persistence, timing priorities and jitter. No specific STM32 device approved yet.

### Planned Section 17: Validation and acceptance tests

**TO DEVELOP.** ROM probing/status and commands, TACH verification during ordinary 800K reads, five standard sector-count classes, SYNC6 calibration, full GCR read/write/format, boot, electrical jitter, media eject/removal and flash durability.

### Planned Section 18: Open questions and decision log

Future cross-part decision-log consolidation. Current Part 1 research notes remain in the unnumbered appendix below.

## Decisions and change history

*This unnumbered appendix preserves historical Section 17 notes pending their eventual allocation to Section 18.*

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

- **2026-11-09 (new Part 4):** Added [Part 4 / Section 12](06-part-4-serial-gcr-write.md) covering Sony WR **both-edge** behavior, gate/erase timing, ROM data-only sector writes vs formatting, and candidate STM32 capture, alignment and safe persistence, with unresolved splice and flash atomicity made explicit.
- **2026-11-09 (new Part 3):** Added [Part 3 / Section 11](06-part-3-media-architecture.md) for the common raw-GCR, SYNC6 timing, logical-track, metadata and TACH contract. Shifted serial writing to Part 4 / Section 12, and deferred headings to 13–18. Removed physical-zone implementation language from the STM32 state model without removing host-visible TACH requirements.
- **2026-11-09 (Section 9):** Developed Macintosh Plus disk-insertion, ROM VBL polling and first-access state; corrected the empty-drive `/WRTPRT=0` semantics and documented Sony's special empty-but-injected `/CSTIN` case; specified software eject timing, ROM post-eject suppression, logically ejected/physically retained cartridge state, qualification, write protection, flash-durability constraints and tests. Section 9 remains open for mechanical and storage-design implementation decisions.
- **2026-11-06 (Section 8):** Added motor-control versus IWM drive-enable distinction, five authentic RPM zones and derived 60-pulse/revolution TACH rates, the ROM's **unexpected 800K ReSeek tach-speed validation**, /READY conditions and timing diagrams, automatic-insertion caveats, and STM32 timer/ready state model with validation tests. No physical spin-up or seek delays added as requirements.
- **2026-11-06 (Section 7):** Expanded the ROM-driven one-track seek and recalibration operations, /STEP acknowledgement versus /READY, /TK0 status, original step/access timing diagrams and limits, 80-track geometry, immediate virtual stepping, STM32 state model and validation matrix. Documented hardware-only physical timing as *limits*, not required artificial delays.
- **2026-11-06 (Section 6):** Added the eight-pattern /DIRTN, /STEP, /MOTORON and EJECT command table, LSTRB rising-edge protocol, both 1985/1986 timing references (including an explicit T4/T5 discrepancy), actual Plus ROM issue sequences, short 800K eject handling, and an STM32 command decoder/test model. Sections 5 and 7–16 remained for analysis at the time of this historical note.
- **2026-11-06:** Created Document 06: operation-first contents and a detailed, source-referenced **status-read** section. Correctly distinguished ROM address notation from project notation, identified ROM startup probes, and left unverified/read-unassigned outputs explicitly open. All subsequent operations remain scaffolded.


- **2026-11-09 (document split):** Renamed original Document 06 to **06 — Part 1**, retaining full Sections **1–9**, moving detailed Section **10** to [06 — Part 2](06-part-2-serial-gcr-read.md), and assigning Section **11** to future **06 — Part 3**. Kept earlier Sections 11–17 as a deferred roadmap at the time (subsequently renumbered when Part 3 was inserted).
