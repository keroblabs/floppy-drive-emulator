# Macintosh Plus Floppy-Disk Emulator — Project Overview

**Status:** Agreed project baseline; shared GCR/media decisions updated 9 November 2026  
**Primary target:** Macintosh Plus M0001AP  
**Emulated device:** Sony MFD-51W 800K floppy-disk drive mechanism  
**Primary connection:** Macintosh Plus internal 20-pin drive interface

## 1. Project objective

Develop a standalone electronic replacement for the Sony MFD-51W floppy-disk mechanism used in the Macintosh Plus. From the Macintosh's perspective, the replacement must expose the same **host-visible electrical interface and logical behaviour** as the original drive.

The Macintosh's existing **IWM (Integrated Woz Machine)** remains the floppy-disk controller. The emulator is a **drive replacement, not an IWM/controller replacement**. The goal is to operate with the original Macintosh hardware, ROM and software without requiring patches, additional drivers or modifications to the computer.

Compatibility is defined at the **Sony drive interface**, not at the Macintosh filesystem or application layer. A compatible host controller should not need to know whether it is communicating with the original mechanism or this emulator.

## 2. Foundational design decisions

### 2.1 Electrical and logical emulation, not mechanical simulation

The emulator reproduces the externally observable behaviour required by the IWM, including:

- Input and output signal levels, polarity and safe electrical interfacing.
- Drive control and addressed status/control responses.
- Logical track position, head/side selection, media presence, write protection and eject state.
- Correctly timed serial read-data output and write-data reception, with Apple GCR handling.
- Disk-image reads and persistent writes to removable solid-state media.

**We will not simulate the original mechanical processes for their own sake.** There is no requirement to model real head motion, seek settling, spindle acceleration, rotational inertia, acoustic effects or mechanically realistic delays. For example, a step command may update a digital track-position state immediately.

**Interface timing still matters.** The emulated drive must preserve GCR bit-cell/data-transition timing, input/output setup and hold requirements, command sequencing and any other timing that the IWM actually depends on. A mechanical-origin delay will only be retained if tests or source documentation establish that it is necessary for interoperability.

The distinction is deliberate: **omit mechanical realism; preserve host-visible compatibility**.

### 2.2 Initial host and interface

- Initial reference machine: **Macintosh Plus M0001AP**.
- Initial hardware target: **internal 20-pin floppy-drive interface**.
- Initial mechanism reference: **Sony MFD-51W**; confirm the exact hardware revision and wiring against authoritative documentation and the target machine before designing a PCB.
- Primary disk operation: **800K double-sided**; **400K single-sided** compatibility is also an objective.
- The emulator must support both reading and writing, including booting from compatible disk images.

The Macintosh external **DB-19** interface is a possible later extension using a correctly engineered adapter or interface variant. Do not assume all Sony MFD-51W-family revisions and harnesses are pin-for-pin interchangeable. In particular, revision-specific signal use and any supply rails must be verified before hardware connection.

### 2.3 Removable media concept

The preferred storage design is a **3D-printed, 3.5-inch floppy-shaped cartridge** containing an **SPI flash device** and electrical contacts that mate with the emulator.

- The cartridge provides removable, non-volatile disk-image storage.
- The emulator translates drive-level reads and writes into storage operations; the Macintosh need not be aware of SPI flash.
- An SD-card media path is **not part of the current preferred concept**.
- **16 Mbit / 2 MiB SPI NOR** is the current capacity target. **Standard tracks use structured encoded-GCR records (32 metadata bytes preserving the encoded address/data headers + 703 GCR data/checksum bytes per sector)**; complete cyclic tracks, SYNC6 and gaps are reconstructed in RAM. A **RAW_TRACK identifier is reserved for possible future nonstandard support**, but RAW_TRACK read/write is **not required in the first firmware**. The design requires **at least three >=12-KiB complete-track RAM buffers**, one global timing calibration, logical track descriptors and extensible, migratable metadata.
- Exact flash allocation/ABI, write persistence, power integrity, contacts and safe removal behaviour remain to be designed; see [06 — Part 3 / Section 11](06-part-3-media-architecture.md).

The desired physical presentation is that of an original Macintosh 3.5-inch floppy drive, with a cartridge inserted through the normal opening.

### 2.4 Chassis and eject mechanics

The project intends to **reuse the original drive chassis or its mounting/form-factor geometry**, preserving the Macintosh's physical appearance.

The Macintosh M0001AP has an emergency manual-eject hole for a paperclip. Its presence is relevant to the future physical design, but **the ejection mechanism has not been selected**. Reuse of existing parts, 3D-printed additions, sensors and actuators are candidates, not settled requirements.

Keep two concerns separate:

1. **Host-visible eject behaviour:** the emulator must implement the drive's logical response to the Macintosh's eject command.
2. **Physical cartridge handling:** insertion, retention, release and possible manual-eject operation belong to the mechanical design and need not duplicate the original drive mechanism.

Detailed chassis, cartridge and contact engineering will be developed in a separate workstream.

## 3. Functional architecture

```text
Macintosh Plus M0001AP
  MC68000 / ROM
        |
       IWM
        |
  internal 20-pin interface
        |
  +-------------------------------+
  | Sony MFD-51W drive emulator   |
  |                               |
  | Electrical host interface     |
  | Drive state / status logic    |
  | GCR read/write data engine    |
  | Disk-image and write handling |
  | SPI flash cartridge interface |
  +-------------------------------+
        |
  Removable 3D-printed cartridge
  with SPI flash and contacts
```

This is a **functional decomposition only**. **STM32 is the preferred implementation platform**, but no specific part, peripherals, electrical buffer or interrupt/DMA architecture is yet committed. The design requires deterministic GCR pulse output and write-transition capture; ordinary GPIO polling has not been validated.

## 4. Explicit project boundaries

**In scope:** faithful host-facing MFD-51W emulation for the Macintosh Plus, 400K/800K disk operations, removable SPI flash media, supporting electronics and the chassis/cartridge work needed to use it.

**Out of scope for this repository's initial project:**

- Replacing, reproducing or emulating the Macintosh's IWM controller.
- Apple IIe or Apple IIGS controller hardware.
- Central Point Universal Disk Controller reproduction.
- ProDOS integration or Apple II SmartPort emulation.
- Changes to or dependencies on the separate **Apple FPGA** project.
- Macintosh 1.44 MB SuperDrive or HD20 hard-disk emulation as initial requirements.

A future dedicated controller in the **Apple FPGA** project might drive this emulator as though it were a physical MFD-51W. That possibility is **not a requirement** for the Macintosh emulator and does not change its scope.

## 5. Engineering approach

1. **Specify the interface:** establish revision-specific connector mapping, voltages, electrical limits, register/control/status behaviour and GCR timing from reliable source documents.
2. **Define the architecture:** choose digital implementation hardware only after identifying functional and timing constraints.
3. **Implement and integrate:** develop the emulator, media storage and separate physical cartridge/chassis subsystem.
4. **Validate on hardware:** compare host-visible behaviour with the original mechanism and test 400K/800K read, write, boot, eject and media-state handling on a Macintosh Plus.

Documentation should distinguish **verified source facts**, **observations or measurements**, **agreed project decisions** and **open design questions**. Unverified assumptions must not become PCB or firmware requirements.

## 6. Repository workspaces and current documentation

- [02 — Macintosh 20-pin interface](02-macintosh-20-pin-interface.md): electrical connections and signal behaviour.
- [03 — MFD-51W versus MFD-51W-03](03-mfd-51w-vs-mfd-51w-03.md): mechanism and revision differences.
- [04 — Chassis and mechanics quick-start](04-chassis-and-mechanics-quickstart.md): mechanical measurement and development.
- [05 — Cartridge contact concepts](05-cartridge-contact-concepts.md): removable-media contact options.
- [06 — Part 1 / Sections 1–9: Host interface and drive operations](06-part-1-drive-interface.md): selector, control/status, seek, TACH, readiness and eject.
- [06 — Part 2 / Section 10: Serial GCR reading](06-part-2-serial-gcr-read.md): waveform, 6-and-2 track structure and freshly formatted track reference.
- **[06 — Part 3 / Section 11: Shared media architecture](06-part-3-media-architecture.md):** agreed structured encoded-GCR sector records plus reserved RAW_TRACK extension, three-buffer cache, one SYNC6 calibration, track descriptors/TACH, 2-MiB allocation budgets, extensible directory and external programming workflow.
- **[06 — Part 4 / Section 12: Serial GCR writing](06-part-4-serial-gcr-write.md):** Sony write gate and both-edge WR capture, 50-SYNC6 candidate format detection, structured sector replacement, asynchronous flash persistence and open power-cut/raw-track write gates.
- **[06 — Part 5 / Section 13: Formatting, geometry and 400K compatibility](06-part-5-format-geometry-400k.md):** 400K/800K sector classes, side/format bytes, preserved interleave/sector identities, complete standard track reconstruction, seam and ROM format/Verify checks.
- **[06 — Part 6 / Section 14: Standard-media error handling](06-part-6-error-handling-standard-media.md):** first-release standard-image-only scope, absent/blank/protected media, interrupted writes/formatting, retries, flash/cache faults and safe removal.
- **[06 — Part 7 / Section 15: Electrical requirements and timing](06-part-7-electrical-timing.md):** Plus internal 20-pin rails, Sony TTL thresholds, RD true high-Z/fast mux, sub-µs GCR pulses, WR gate/edges, LSTRB/STEP/TACH/READY and safe power. Sections 16–18 remain planned.
- `source/`: firmware and/or digital-logic implementation, pending architecture selection.
- `kicad/`: schematics and PCB design.
- `3d-drawing/`: mechanical CAD and cartridge/chassis models.

The documents above began as scaffolding and **are not automatically authoritative** where they differ from this agreed overview. Electrical details and mechanical solutions must be verified and refined in their own workstreams.

## 7. Baseline principle

> **We are building an electrical and logical Sony MFD-51W drive emulator, not a mechanical drive simulator or a Macintosh floppy controller.** The existing IWM must see the signals and behaviour it needs; internal implementation details and unnecessary mechanical delays are free to differ.
