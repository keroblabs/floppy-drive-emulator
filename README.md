# floppy-drive-emulator

Floppy disk drive emulator for Macintosh Plus 1M (M0001AP).

## MFD-51W / MFD-51W-03 documentation pack

This repository now includes a focused documentation scaffold for developing an emulator based on the Sony MFD-51W / MFD-51W-03 chassis, with:

- 3D printed 3.5" cartridge
- cartridge PCB + SPI flash storage (no SD card)
- optional chassis simplification using printed parts, sensors, and servos

Start in `/docs`:

- `/docs/01-overview.md`
- `/docs/02-macintosh-20-pin-interface.md`
- `/docs/03-mfd-51w-vs-mfd-51w-03.md`
- `/docs/04-chassis-and-mechanics-quickstart.md`
- `/docs/05-cartridge-contact-concepts.md`

Working folders:

- `/source` (STM32 firmware workspace)
- `/3d-drawing` (SketchUp 2018 models)
- `/kicad` (KiCad 9 schematic/PCB files)
