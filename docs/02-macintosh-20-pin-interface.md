# Macintosh 20-Pin Interface (MFD-51W Family)

> Use this as a working checklist and verify against your hardware before finalizing the design.

## Practical signal groups

For emulator development, classify the 20-pin harness into functional groups:

1. **Power and return**
   - +5V supply
   - Ground return(s)
2. **Drive select / motor / control lines**
   - drive enable/select-style controls
   - motor enable/control lines
3. **Data path lines**
   - read-data output toward host
   - write-data input from host
   - write-gate / write-enable class signal
4. **Status / handshake lines**
   - track-0 / index / ready-like status signals (model/revision dependent)
5. **Optional or legacy lines**
   - eject-related line(s)
   - PWM-related behavior on some implementations

## Emulator-specific guidance

Because this emulator uses software-controlled eject and custom cartridge media:

- Keep core data/control/status signaling needed by the host logic.
- Mark original eject and PWM-related paths as optional unless needed for compatibility tests.
- If a line is omitted electrically, define firmware behavior that avoids host timeout assumptions.

## Bring-up checklist

- Confirm supply voltage and inrush profile with oscilloscope.
- Capture idle logic levels of all non-power pins before connecting emulator outputs.
- Verify directionality for each active line (host->drive or drive->host).
- Add temporary series resistors or buffers during first prototype bring-up.
