# Macintosh Plus Internal 20-Pin Floppy Interface — MFD-51W Emulator

**Status:** Verified host connector assignment; detailed electrical/timing implementation still under investigation  
**Target host:** Macintosh Plus M0001AP  
**Drive being emulated:** Sony MFD-51W (Apple internal 800K mechanism)  
**Interface:** Internal 20-pin Macintosh Plus floppy-drive connection

> **Scope:** This document specifies the interface **as presented by the Macintosh Plus to an internal MFD-51W replacement**. It is not a generic Apple floppy pinout, nor an assertion that all Sony MFD-51W-family variants use identical PCB connections. Our purpose is electrical and logical compatibility with the existing IWM, *not* reproduction of mechanical drive delays. See [01-overview.md](01-overview.md).

## 1. Primary references and confidence

1. **Apple**, *Guide to the Macintosh Family Hardware*, 2nd edition (1990), Chapter 9:
   - Figure 9-3: internal 20-pin connector orientation, printed page 334.
   - **Table 9-3: Macintosh Plus internal connector assignments**, page 335.
   - Explanatory text following Tables 9-3/9-4, pages 336–337.
   - [Reference PDF](https://vintageapple.org/inside_o/pdf/Guide_to_Macintosh_Family_Hardware_2nd_Edition_1990.pdf).
2. **Apple**, *800K Double-Sided Drive Engineering Requirements Specification*, drawing **699-0452-A** (September 1986):
   - Section 3: Sony drive command/status protocol, electrical characteristics and timing.
   - Connector assignment: sheet 37; variant comparison: **Appendix D, sheet 47**.
   - [Reference PDF](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_specs/669-0452-A_800K_Double-Sided_ERS_Sep86.pdf).

**Verified documentation facts** below refer to these sources. **Emulator requirements** reflect [our agreed scope](01-overview.md). **Open items** must be tested or specified before a schematic is approved.

## 2. Macintosh Plus internal connector: complete pin map

The following is the **Macintosh Plus host-side assignment**, from Apple's Table 9-3, **not** a universal Sony drive-PCB assignment. Direction is relative to the **emulator**. A slash prefix denotes an active-low signal where specified. Consult Apple's Figure 9-3 and verify the physical connector orientation/keying before constructing a cable; a drawn two-row pin sequence alone is insufficient to determine a safe mating orientation.

| Pin | Macintosh Plus signal | Emulator perspective | Function / requirement |
|---:|---|---|---|
| 1 | GND | Power return | Ground |
| 2 | PH0 | Input | State/control address bit; corresponds to Sony **CA0** |
| 3 | GND | Power return | Ground |
| 4 | PH1 | Input | State/control address bit; Sony **CA1** |
| 5 | GND | Power return | Ground |
| 6 | PH2 | Input | State/control address bit / command value; Sony **CA2** |
| 7 | GND | Power return | Ground |
| 8 | PH3 | Input | Register-write strobe; Sony **LSTRB** |
| 9 | −12V | Supply present on host; **do not connect** for original MFD-51W profile | Original internal MFD-51W PCB has **no connection on pin 9** |
| 10 | /WRREQ | Input | IWM programmable write-request/handshake output; corresponds to Sony **/WRTGATE** function |
| 11 | +5V | Supply | +5 V drive power |
| 12 | SEL | Input | Additional register selector; also selects read-data head/side in data mode |
| 13 | +12V | Supply | +12 V drive power present |
| 14 | /ENBL.INT | Input | Active-low internal-drive enable; Sony **/ENBL** |
| 15 | +12V | Supply | +12 V drive power present |
| 16 | RD | **Tri-state output** | Sony **RD**: selected status bit **or** serial read-data stream |
| 17 | +12V | Supply | +12 V drive power present |
| 18 | WR | Input | Serial write-data transitions; Sony **WRTDATA** |
| 19 | +12V | Supply | +12 V drive power present |
| 20 | PWM | Host output; **do not connect** for original MFD-51W profile | PWM for older 400K single-sided mechanisms; original internal MFD-51W PCB has **no connection on pin 20** |

**Critical distinction (confirmed in the two Apple sources):**
- The **Macintosh Plus host** supplies **−12 V on pin 9** and **PWM on pin 20**.
- The **original internal Sony MFD-51W** used in the Macintosh Plus leaves **pins 9 and 20 unconnected** at the drive PCB.
- The **MFD-51W-10** external 800K variant leaves pin 9 unconnected but uses pin 20 differently; the **MFD-51W-03** Apple 3.5 Drive variant has different uses for **both** pins. **Never interchange the variants without a verified interface map.** See [03-mfd-51w-vs-mfd-51w-03.md](03-mfd-51w-vs-mfd-51w-03.md), which requires a separate revision pass.

**No emulator GPIO, power rail or ground may be directly attached to host pin 9.** Host pin 20 must similarly not be assigned an arbitrary GPIO/output. Their treatment must follow the exact original MFD-51W profile unless a future explicitly designed adapter changes the arrangement.

## 3. How drive control and status actually work

This interface is **not** the PC floppy interface: there are **no separate physical track-zero, index, ready, write-protect or eject-status pins**.

### 3.1 Address selection and commands

- **PH0–PH2** are Sony **CA0–CA2**; **SEL** participates in choosing a drive register or status source.
- **PH3 / LSTRB** latches a control command. Under the Sony specification, the level of **CA2** is written into the addressed latch on the **rising edge** of LSTRB.
- Addressable drive functions include **direction**, **step**, **motor on/off** and **eject**.
- **SEL is not only a dedicated head-select pin:** its meaning depends on the register/data selection.
- **/ENBL.INT** qualifies drive communication. When the drive is disabled, the Sony specification requires **RD to be high impedance**, not actively driven high or low. Drive-control latch defaults/reset behaviour also matter.

The emulator implements the appropriate digital state transitions without physically moving a head or spinning a motor.

### 3.2 Multiplexed RD line

Sony's Section 3.1, Table 1 shows that **SEL, CA2, CA1 and CA0** select what is returned on the **single RD output**. The selections include:

- Read-data streams for **head 0** and **head 1**.
- **Track zero**, **ready**, **tachometer**, **drive present**, **cartridge/disk present**, **write-protect** and **drive-type/revision** status.
- Feedback for command/control latches such as direction, step, motor and eject.

These are **logical statuses selected onto RD**, not individual connector pins. The emulator must return correct values and polarity for the original MFD-51W; details belong in a dedicated register/state-machine specification.

### 3.3 Write path

- **WR / WRTDATA** carries the serial write-data transitions from the IWM.
- **/WRREQ** is generated by the IWM and participates in write handshaking; the Sony drive specification describes the corresponding functional input as **/WRTGATE**.
- The emulator must apply the proper enabled/write-gate/write-protect conditions before committing changes to stored media.
- Data capture, GCR interpretation and persistence are independent of any physical write head.

**Caution:** Signal names from the host and drive documents are not always identical. The table identifies functional correspondences; exact active levels and edge timing must be validated from the relevant sections before GPIO or HDL implementation.

## 4. Electrical requirements and constraints

### 4.1 Power

For the Macintosh Plus 20-pin connection, **+5 V, +12 V and −12 V are all present** (though the original MFD-51W leaves drive pin 9, −12 V, unconnected).

Apple's Chapter 9 states a **combined load ceiling across all devices connected to a Macintosh Plus** of:
- **600 mA at +12 V**;
- **700 mA at +5 V**;
- **10 mA at −12 V**.

Apple also quotes an original 800K drive load of **600 mA at +12 V** and **360 mA at +5 V**. These are **historical host/loading specifications, not target currents for our electronic replacement**. Our design should aim for a materially lower load and must include the total-machine power budget, protection, transients and safe startup in its power design.

We have **not** decided which supplied rails the emulator circuitry will actually consume. **Do not repurpose or short unused rails**.

### 4.2 Logic and direction

- Treat the host lines as **5 V-era logic**. The Sony engineering specification, Section 3.3, contains relevant input thresholds and RD output-drive limits; select suitable interfacing only after checking them against the actual Macintosh host circuit and the chosen implementation technology.
- Inputs to the emulator: **PH0, PH1, PH2, PH3, SEL, /ENBL.INT, /WRREQ, WR**.
- The sole drive-to-host digital output in the main 800K interface is **RD**, which carries both selected statuses and read data.
- RD must be **tri-stated whenever drive enable is inactive**, according to the Sony requirement.
- Unused pin 9 (−12 V) and pin 20 (PWM) must be safely isolated in the **internal MFD-51W** implementation.
- Do not select MCU/FPGA I/O voltage, buffers, pull-ups, termination or power-domain circuitry without first validating the electrical characteristics.

### 4.3 Interface timing versus mechanical timing

**Required:** valid command strobe/address setup and hold; drive-enable qualification; correct RD multiplexing/tri-state behaviour; GCR read-data transition timing; reliable write-transition capture; and any readiness/command sequencing the host demonstrably requires.

**Not inherently required:** recreating real mechanical head-seek time, motor spin-up, physical rotational latency or other delays merely to imitate the old mechanism. A digital track counter can respond immediately **provided host-visible protocol expectations remain correct**.

The Sony specification includes original mechanical timing limits; these are **reference characteristics**, not automatically requirements for our emulator. Any necessary compatibility delay must be justified by documentation, observed IWM behaviour or a test.

## 5. Remaining electrical and implementation questions (Section 15 developed)

**[DEVELOPED]** Numeric Sony DC levels, RD tri-state/response, GCR pulse width and write/strobe/control timing are now specified in **[Document 06 — Part 7 / Section 15](06-part-7-electrical-timing.md)**. **[Part 8 / Section 16](06-part-8-stm32-implementation-architecture.md)** now describes candidate STM32 timers, buffers, fast RD interface and flash/power architecture without approving a final circuit. The host-specific pin map in this document remains authoritative. The points below are remaining **bench and circuit implementation tasks**; older research wording is retained where detailed physical values have now been established.

1. Confirm **physical mating orientation**, keying, cable pin-1 identification and the exact M0001AP internal harness before prototype connection.
2. Build the full **RD selector table and write-latch map**, including power-up, disable and media-change state transitions.
3. **Verify/implement** the numeric electrical limits already specified in [Section 15](06-part-7-electrical-timing.md#153-dc-logic-levels-loading-and-rd-output-drive), and measure actual Plus host termination/pull-ups, RD load and 5-V-safe interface margin.
4. **Bench-check** the Section 15 RD/WR waveform and gate timing against the target IWM, including total buffer delay, pulse widths, gate phase and tolerances; protocol values are now documented.
5. **Validate** READY/TACH and other host-visible statuses against Section 15 and the ROM without adding arbitrary mechanical wait time.
6. Design reset, power loss, insertion/ejection, write-protection and **SPI flash cartridge hot-removal** semantics without exposing invalid states to the IWM.
7. Validate the selected physical-drive revision and reconcile differences with the **MFD-51W-10 / MFD-51W-03** documents before considering any external DB-19 support.

No PCB layout or firmware assignment should be based on an unresolved item.

## 6. Safe bring-up plan

1. Photograph the target host connector and harness; verify pin numbering and continuity **with power off**.
2. Confirm supply rails and idle levels using suitable instrumentation and safe probing procedures; do not probe a live compact Macintosh's high-voltage analogue/CRT areas.
3. Verify host-line direction, state transitions, and the RD tri-state/selection scheme **on an isolated, current-limited bench prototype** before attaching it to the Macintosh.
4. Exercise command-register selections and logical statuses with simulated host waveforms.
5. Verify precise read/write serial timings and protection against contention/overvoltage.
6. Only then connect to the Macintosh Plus, with the host power/load budget and all connector voltages checked.

**Baseline:** The emulator must look electrically and logically like the original **internal Sony MFD-51W**, not like a generic floppy mechanism. No mechanical simulation is required except where a host-visible timing contract demands it.
