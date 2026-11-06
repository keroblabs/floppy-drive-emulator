# MFD-51W vs MFD-51W-03 Cable/Interface Notes

## Why this matters

Both drives are close mechanically, but cable wiring or optional line use can differ by revision and host implementation. This emulator can intentionally de-scope signals not required by the new architecture.

## Working difference table

| Topic | MFD-51W | MFD-51W-03 | Emulator decision |
|---|---|---|---|
| Main chassis compatibility | Baseline chassis | Generally similar chassis family | Reuse and simplify where practical |
| Eject line behavior | Present in legacy control path | Present but may differ in handling | **Not required** (software-controlled eject) |
| PWM-related control | May be used in some original behavior | May be variant-dependent | **Not required** unless compatibility tests demand it |
| Cable pin assignments | Revision dependent | Revision dependent | Validate per harness and map only required signals |

## Recommended workflow for cable differences

1. Photograph connector keying and wire colors for each candidate drive.
2. Continuity-map each harness pin to drive PCB pads.
3. Build one canonical project pin map (`host_pin -> emulator_signal -> firmware GPIO`).
4. Keep an adapter board option in case MFD-51W and MFD-51W-03 harnesses differ.

## Decision for this project

- Required: power, ground, read/write/control/status lines needed by host protocol.
- Optional/omitted by default: eject pin and PWM-specific line(s).
