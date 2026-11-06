# Sony MFD-51W, MFD-51W-10 and MFD-51W-03 — Interface Variants and Emulator Decisions

**Status:** Revision differences documented; **MFD-51W profile approved** as the initial emulator target (6 November 2026)  
**Reference computer:** Macintosh Plus M0001AP  
**Initial connection:** Macintosh Plus **internal 20-pin** floppy-drive harness

> This is a **mechanism and electrical-interface comparison**, not a proposal to emulate three different drives immediately. The agreed project is an **electrical and logical replacement for the original internal Sony MFD-51W**. Refer to [01-overview.md](01-overview.md) for project scope and [02-macintosh-20-pin-interface.md](02-macintosh-20-pin-interface.md) for the host connector and control signals.

## 1. Sources and evidence

### Primary sources

1. **Apple, 800K Double-Sided Drive Engineering Requirements Specification**, drawing **699-0452-A**, September 1986:
   - **Appendix D, sheet 47**: the three Sony PCB variants, intended Apple products, and pin-9/pin-20 differences.
   - **Section 3.9, sheet 37**: 20-pin drive-interface connector assignment for the Apple 3.5 Drive variant specified by that document.
   - [Scanned engineering specification (PDF)](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_specs/669-0452-A_800K_Double-Sided_ERS_Sep86.pdf).
2. **Apple Technical Info Library #1605, “Apple 800K Drive Mechanism and Cable Compatibility”**, revised 16 September 1988:
   - Compatibility matrix and **required Macintosh internal cable part numbers**.
   - [Technical Info Library article (PDF)](https://savagetaylor.com/TIL/TIL01605.pdf).
3. **Apple, Guide to the Macintosh Family Hardware**, 2nd edition (1990), Chapter 9:
   - Macintosh Plus **host-side** 20-pin assignments and floppy interface description.
   - [Hardware guide (PDF)](https://vintageapple.org/inside_o/pdf/Guide_to_Macintosh_Family_Hardware_2nd_Edition_1990.pdf).

**Evidence classification:** Information attributed to these publications is *documented*. Engineering choices below are *agreed project decisions*. Connector-specific electrical measurements, cable continuity and detailed timing not yet performed are explicitly *open verification tasks*.

## 2. Three Sony 800K mechanism variants

Apple's specification, Appendix D, states that the three Sony 800K variants were **mechanically identical in its qualified configuration**. Their important differences were on the **20-pin drive PCB connector** and associated signal use.

| Characteristic | MFD-51W | MFD-51W-10 | MFD-51W-03 |
|---|---|---|---|
| Historical intended application | Internal Macintosh Plus; also UniDisk 3.5 mechanism | Macintosh external 800K drive | Apple 3.5 Drive |
| Sony/Apple engineering PCB variant | Apple 699-0321 | Apple 699-0451 | Apple 699-0452 |
| Apple service mechanism grouping | 661-0305 | 661-0305 | 661-0345 |
| 20-pin **pin 9 at drive PCB** | **N.C.** | **N.C.** | **EJECT-related** signal |
| 20-pin **pin 20 at drive PCB** | **N.C.** | **ANDed with RD DATA** | **/CSTOUT** |
| Mechanical architecture | Shared 800K mechanism family | Shared family | Shared family |
| Emulator project role | **Primary, required** | Reference / possible future variant | Reference / possible future variant |

**N.C. means no connection at the drive PCB**, not that the corresponding Macintosh host pin is absent or unpowered.

**Signal naming caution:** Appendix D labels pin 9 on the MFD-51W-03 as **EJECT**; Section 3.9 labels the connector's eject signal **/EJECT**. Treat its precise polarity, drive direction and interaction with host wiring as **revision-specific details requiring the complete source context and verification**. Do not implement a pin-9 interface based on an abbreviated table.

The pin-20 output on the MFD-51W-10 is **not equivalent to a simple unused PWM input**, and the MFD-51W-03's **/CSTOUT** is not the Macintosh Plus host's PWM signal.

### 2.1 The host connector is a different specification layer

On the **Macintosh Plus internal host connector** (Apple hardware guide), the relevant pins are:

| Macintosh Plus host pin | Host function | Original internal MFD-51W PCB connection |
|---|---|---|
| 9 | **−12 V supply** | **N.C.** |
| 20 | **PWM output** (legacy motor-speed control) | **N.C.** |

This is the first non-negotiable safety rule for our original-MFD-51W replacement:

> For the **initial internal MFD-51W profile**, pins **9 and 20 must remain safely isolated** from emulator circuitry, as they are on the original MFD-51W PCB. Never connect host −12 V to logic, an eject input or a ground return, and never reinterpret host PWM as another variant's drive output.

Other pins in the 20-pin interface still carry power, drive-control signals and multiplexed data/status. Their implementation belongs to [Document 02](02-macintosh-20-pin-interface.md); pin 9 and pin 20 are **not** the complete electrical specification.

## 3. Compatibility depends on the cable — not merely on the mechanism

The variants **cannot be assumed pin-for-pin interchangeable when plugged into the same internal cable**.

Apple Technical Info Library #1605 explicitly documents both mechanism groups as usable in an internal Macintosh **when the correct cable is selected**:

| Mechanism fitted internally | Apple service mechanism part | Required internal cable |
|---|---|---|
| **MFD-51W / MFD-51W-10** | **661-0305** | **590-0167**, red-striped cable |
| **MFD-51W-03** | **661-0345** | **590-0437**, yellow-striped cable |

**Important conclusion:** The MFD-51W-03 **can** be an internal Macintosh drive with the correct cable. It is **not** safe to infer that its PCB connector can simply be substituted into the original MFD-51W red-striped harness. Pin 9 is the most obvious danger because the host carries −12 V there while the -03 drive uses that position for an eject-related signal.

The Apple compatibility note also lists both mechanism groups as usable inside the Macintosh **external 800K drive enclosure** and **UniDisk 3.5 enclosure** (with their appropriate associated electronics), but identifies the older 661-0305 mechanism group as **not** suitable for the Apple 3.5 Drive enclosure. These are **service replacement rules**, not a universal adapter circuit or permission to hot-plug a mechanism.

**Cable colour is a historical identifier only.** For actual hardware, confirm Apple cable part numbers, pin orientation and continuity, because colour, modifications and replacement harnesses alone are insufficient guarantees.

## 4. Approved emulator decisions

### 4.1 Reproduce the MFD-51W, not all three variants

- The initial emulator is a **Sony MFD-51W electrical/logical replacement**, presented to the Macintosh Plus IWM through the **internal 20-pin connector**.
- **MFD-51W-10 and MFD-51W-03 are research references** and possible later compatibility profiles, **not initial hardware requirements**.
- There is **no decision** to use jumpers, software auto-detection, or a universal connector profile to support all variants.
- The external Macintosh **DB-19** interface is a **future possibility**, requiring a separately validated wiring/electrical design. DB-19 pin numbers and signal mapping must not be confused with the internal drive PCB's 20-pin numbering.

### 4.2 Emulate host-visible behaviour, not mechanical realism

- Respond to the actual IWM control/address interface: **PH0–PH3, SEL, /ENBL.INT**, write controls and serial data.
- Provide the **RD** output's selected logical statuses and properly timed GCR data, including its required high-impedance state when disabled.
- Preserve host-visible **disk-present, write-protect, track-zero, ready, motor and eject** semantics and appropriate control-state transitions.
- **Eject-command interpretation is mandatory**: the fact that original MFD-51W **pin 9 is N.C.** does **not** mean software eject can be ignored. Eject is also controlled through the Sony **addressed/latch command protocol**, independently of the -03 variant's extra pin-9 signal.
- Emulate the track position, heads and media electronically. **Do not simulate physical head movement, spin-up, track settling or rotational delays merely for realism.** Keep only timing actually required for electrical protocol, GCR exchange or demonstrated host compatibility.

### 4.3 Media and physical design stay independent of the connector variant

- Current preferred media: **removable 3D-printed 3.5-inch floppy-shaped cartridge with SPI flash and mating electrical contacts**, not an SD-card path.
- The original drive **chassis/form factor** is the mechanical reference; the detailed insertion, retention, release and paperclip/manual-eject mechanisms are **not yet selected**.
- Logical disk ejection must be reflected to the IWM even though the physical release mechanism may be entirely different from Sony's original electromechanical design.
- Chassis geometry and removable-media mechanics are covered separately by [Document 04](04-chassis-and-mechanics-quickstart.md) and [Document 05](05-cartridge-contact-concepts.md).

### 4.4 Do not choose implementation hardware prematurely

No STM32, FPGA, CPLD, buffer family, programmable-I/O layout or universal adapter board is committed. The drive protocol and timing specification must be sufficiently understood before selecting the architecture.

This project is **independent of the Apple FPGA project**; an eventual Apple IIe/ProDOS controller, if pursued, belongs to that separate project.

## 5. Safety and interface-development rules

1. **Target one exact profile first:** Macintosh Plus M0001AP + original **MFD-51W** + matching internal harness.
2. **Do not rely on connector shape alone:** a mechanically mating 20-pin plug is not evidence of electrical interchangeability.
3. **Keep pins 9 and 20 isolated** for the original MFD-51W profile. Document this explicitly in the schematic and check it during bench tests.
4. Verify the host connector's **+5 V, +12 V, −12 V and ground positions**, plus pin-1 orientation, before applying power. Use [Document 02](02-macintosh-20-pin-interface.md) as the current host-map reference, subject to physical verification.
5. Maintain correct **signal direction and tri-state handling** so the emulator cannot drive against an active host output.
6. Any future **-10/-03 compatibility** must have its **own complete electrical map**, adapter/connector treatment and validation. Do not add jumper options based only on the two differing pins.
7. Keep **logical host protocol** and **cartridge mechanics** separate: an alternate physical eject solution never removes the requirement to honour the host's eject request and report cartridge state correctly.
8. **Never hot-plug** the Macintosh floppy harness or drive electronics; removable SPI flash cartridge hot-insertion/removal is a separate design problem requiring its own safe-contact/power/transaction strategy.

## 6. Open verification items

| Item | Why it remains open |
|---|---|
| Confirm exact Sony model and PCB marking of any drive obtained | The reference Macintosh Plus may have a replacement mechanism rather than its original model |
| Verify original and candidate cable continuity, pin-1 marking and connector orientation | Establish safe mapping independent of cable colour |
| Extract full Sony read-register selection and command-latch tables | Required to implement all MFD-51W host-visible status and eject functions |
| Validate signal polarity, RD output/tri-state conditions and DC thresholds | Required for schematic design |
| Establish GCR read/write electrical timing and any compatibility-critical status sequencing | Required for choosing MCU/FPGA logic |
| Examine the functional use of MFD-51W-10 pin 20 and MFD-51W-03 pins 9/20 if expansion is ever approved | Prevent unsafe assumptions when considering adapters |
| Decide how physical insertion/ejection updates logical media state | Coordinate with separate cartridge/chassis workstream |
| Verify MFD-51W-03 eject pin polarity and cable handling using detailed schematics/service notes before any hardware experiment | Avoid connecting an eject-control circuit to the Macintosh Plus host's −12 V rail |

**Decision baseline:** Build and validate an **internal MFD-51W emulator first**, with pins 9 and 20 unconnected at the emulator interface and **full host-visible register/data/eject behaviour**. Mechanism revision compatibility remains a documented future possibility, not an initial implementation commitment.
