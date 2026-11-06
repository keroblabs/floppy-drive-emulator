# MFD-51W / MFD-51W-03 Emulator Overview

## Project objective

Reuse the MFD-51W / MFD-51W-03 mechanical drive chassis to build a floppy emulator that uses:

- a custom 3D printed 3.5" cartridge
- an internal cartridge PCB with SPI flash
- firmware-controlled eject behavior (no dependency on original eject pin behavior)

## Scope boundaries

- No SD card media path in this concept.
- Mechanical simplification is allowed where needed.
- Additive parts (3D printed), servo-actuated elements, and sensors are in scope.

## Deliverables scaffold in this repository

- Interface notes and pin mapping: `02-macintosh-20-pin-interface.md`
- MFD-51W vs MFD-51W-03 cable differences: `03-mfd-51w-vs-mfd-51w-03.md`
- Chassis quick-start + measurement capture guide: `04-chassis-and-mechanics-quickstart.md`
- Cartridge-to-emulator contact concepts: `05-cartridge-contact-concepts.md`

## Validation-first note

Some detailed values vary by drive revision and cable harness. Treat this documentation as an implementation starter and verify electrically/mechanically against your exact units before final PCB and firmware lock.
