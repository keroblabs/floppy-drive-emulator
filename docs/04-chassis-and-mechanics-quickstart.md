# Chassis and Mechanics Quick-Start

## Objective

Capture only the dimensions and constraints needed for rapid prototype fit of:

- emulator board mounting
- cartridge insertion path
- sensor and servo placement

## Minimal measurement set

Measure and record (mm):

1. Chassis envelope (W/H/D)
2. Front bezel opening (W/H) and depth to first internal obstruction
3. Cartridge travel length from entry to seat point
4. Existing screw boss locations and thread type
5. Spindle/clamp region clearance for new cartridge internals
6. Eject linkage path (if reusing any portion mechanically)

## Recommended coordinate system

- Origin: front-left-bottom corner of chassis interior
- X: left -> right
- Y: bottom -> top
- Z: front -> rear

Use the same coordinate system for CAD and KiCad mechanical layers to reduce integration mistakes.

## Servo/sensor integration notes

- Prefer normally-open optical or Hall sensor inputs with debounce in firmware.
- Keep servo mounts slotted for ±1 mm adjustment.
- Add hard end-stops in printed parts so servo torque does not load PCB contacts.
