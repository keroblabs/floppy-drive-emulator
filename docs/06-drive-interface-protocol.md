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

**Architecture caution:** ROM pulses are microseconds long. The final STM32 implementation must capture **LSTRB edges** and handle RD/status changes deterministically; a slow, occasionally scheduled polling loop is not an approved implementation. Timer/input-capture, external logic or interrupt-assisted buffering are candidates for Section 15.

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

- **2026-11-06 (Section 6):** Added the eight-pattern /DIRTN, /STEP, /MOTORON and EJECT command table, LSTRB rising-edge protocol, both 1985/1986 timing references (including an explicit T4/T5 discrepancy), actual Plus ROM issue sequences, short 800K eject handling, and an STM32 command decoder/test model. Sections 5 and 7–16 remain for later analysis.
- **2026-11-06:** Created Document 06: operation-first contents and a detailed, source-referenced **status-read** section. Correctly distinguished ROM address notation from project notation, identified ROM startup probes, and left unverified/read-unassigned outputs explicitly open. All subsequent operations remain scaffolded.
