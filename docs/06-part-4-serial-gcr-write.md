# Sony MFD-51W Drive Interface Protocol — Part 4: Serial GCR Writing

**Document:** 06 — Part 4  
**Protocol section:** **12 — Serial GCR write-data capture, format, track replacement and persistence**  
**Status:** ROM/source-grounded write specification and proposed STM32 processing model; exact bit-phase, gate/end behavior and flash transaction design await real-interface validation  
**Target:** Macintosh Plus M0001AP, original IWM and ROM `.Sony` driver, emulated Sony MFD-51W 800K drive  
**Date:** 9 November 2026

> **Document sequence:** [Part 1 / §§1–9](06-part-1-drive-interface.md) establishes the drive's host interface, controls/status, readiness and TACH; [Part 2 / §10](06-part-2-serial-gcr-read.md) establishes reading and the exact freshly formatted GCR stream; [Part 3 / §11](06-part-3-media-architecture.md) locks the common raw-track, calibration and SPI cartridge principles. **This Part 4 / §12** specifies serial writing. Future §13 covers further format/geometry/400K cases, §14 fault sequences, §15 electrical integration, §16 STM32 hardware design, §17 validation, and §18 decisions.
>
> **Evidence labels:** **[SONY]** original Sony/Apple 800K drive electrical specification; **[IWM]** Apple IWM/SWIM hardware reference; **[ROM]** actual Plus ROM; **[DECISION]** project agreement; **[PROPOSED]** candidate engineering method; **[OPEN]** must be resolved/tested before claiming conformance. A ROM implementation detail is **not** proof of the timing of an external pin.

## Contents

- [12.1 Scope, physical pins and responsibilities](#121-scope-physical-pins-and-responsibilities)
- [12.2 A critical asymmetry: WR level transitions vs RD pulses](#122-a-critical-asymmetry-wr-level-transitions-vs-rd-pulses)
- [12.3 Write-gate qualification and source timing](#123-write-gate-qualification-and-source-timing)
- [12.4 Macintosh IWM write state, byte handshake and underrun](#124-macintosh-iwm-write-state-byte-handshake-and-underrun)
- [12.5 The ordinary sector-write ROM path](#125-the-ordinary-sector-write-rom-path)
- [12.6 Exact normal sector data-field stream](#126-exact-normal-sector-data-field-stream)
- [12.7 Recovering bits and phase from WR edges](#127-recovering-bits-and-phase-from-wr-edges)
- [12.8 Identifying the target sector from virtual read position](#128-identifying-the-target-sector-from-virtual-read-position)
- [12.9 Read-to-write handoff and write splice](#129-read-to-write-handoff-and-write-splice)
- [12.10 Whole-track formatting and first-use blank media](#1210-whole-track-formatting-and-first-use-blank-media)
- [12.11 SYNC6 automatic calibration during formatting](#1211-sync6-automatic-calibration-during-formatting)
- [12.12 RAM buffers, write acceptance and flash persistence](#1212-ram-buffers-write-acceptance-and-flash-persistence)
- [12.13 A proposed STM32 write state machine](#1213-a-proposed-stm32-write-state-machine)
- [12.14 Timing and buffer budget](#1214-timing-and-buffer-budget)
- [12.15 Verification/acceptance matrix](#1215-verificationacceptance-matrix)
- [12.16 Decisions and remaining engineering gates](#1216-decisions-and-remaining-engineering-gates)

---

## 12. Operation: serial GCR writing

### 12.1 Scope, physical pins and responsibilities

**[DECISION]** The emulator implements **drive-level recording**, not a replacement IWM or a logical “write sector N” protocol. The host **does not send a sector number command** to the Sony drive. The original Macintosh ROM first locates a target sector by **reading** its address field, then supplies encoded serial write data to the IWM. The drive only sees the selected side/track state, **write enable** and **WR electrical transitions**.

| Signal | Internal 20-pin connector | Direction at emulator | Function |
|---|---:|---|---|
| **/ENBL.INT** | 14 | input | Selected drive enabled low |
| **/WRREQ** | 10 | input | IWM's active-low write-request output; Sony **/WRTGATE** function |
| **WR** | 18 | input | IWM serialized **WRTDATA level transitions** |
| **RD** | 16 | output, tri-state when disabled | Address/data read pulses, status or TACH according to selector |
| **CA0/CA1/CA2** | 2/4/6 | inputs | Select status/control/read data; PH0–PH2 |
| **SEL** | 12 | input | VIA-driven head/data-source selection |
| **LSTRB** | 8 | input | **Separate** command-latch strobe, not the sector write gate |

Pins **9** and **20** remain **electrically unconnected**. The 800K drive internally implements erase timing; the emulator does not receive a separate host **/ERASE** pin and must not invent one.

**[SONY]** Apple *800K Double-Sided Drive Engineering Requirements Specification*, drawing 699-0452-A, [§§3.2.5–3.2.6, sheets 20–21](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_specs/669-0452-A_800K_Double-Sided_ERS_Sep86.pdf), states that recording is qualified by **/WRTGATE low**, **/ENBL low** and media not being write protected. `WR` carries physical transitions. [Document 02](02-macintosh-20-pin-interface.md) maps Sony's signal names to the Macintosh 20-pin interface. **[OPEN]** Exact pin-level gate behavior on the target Macintosh/IWM revision, including underrun deassertion and enable ordering, remains to be captured.

### 12.2 A critical asymmetry: WR level transitions vs RD pulses

**[SONY — important]** The original electrical specification says that **each change in level** on WRTDATA causes one flux transition to be recorded. **Both rising and falling WR edges matter.** The signal is **not** a series of identical sub-microsecond active-low pulses.

By contrast, the drive's **RD** output delivers a narrow pulse for each recovered transition, with pulse width specified separately in [Part 2 / §10.4](06-part-2-serial-gcr-read.md#104-sony-rd-waveform-and-source-timing).

~~~text
Serial encoded bit cells:   1 1 1 0 0 1 0 1 ...
                           | | |     |   |
WR input:                  _|‾|_|‾‾‾‾‾|___|‾ ...
                            ↑ ↓ ↑     ↓   ↑
                          Every WR edge = recorded 1
                          No edge in a bit cell = recorded 0

RD output:                 short pulses at those same *transition times*,
                           using RD-specific pulse width and polarity.
~~~

The schematic is conceptual, not an exact polarity/phase drawing. **Do not decode WR by sampling its voltage as if high means '1' and low means '0'; do not capture only rising edges.** The absolute starting voltage level is not meaningful as a data bit; **changes** are.

**[SONY]** [§3.4.5.2, sheet 30](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_specs/669-0452-A_800K_Double-Sided_ERS_Sep86.pdf) specifies the WRTDATA transition spacing as **2, 4 or 6 µs nominal**, with **T1–T2 ≤50 ns** shown for the crossing/edge-duration relationship in the source figure. Do not reinterpret this as an RD pulse-width specification. The STM32's front end should accept properly level-shifted **both-edge timestamps**, measured at the actual connector.

### 12.3 Write-gate qualification and source timing

**[SONY]** [§3.4.5.1, sheet 29](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_specs/669-0452-A_800K_Double-Sided_ERS_Sep86.pdf) presents the following relationships in the source diagram. **Qualifiers must not be mixed up**:

| Figure label | Original published number | Interpretation / implementation handling |
|---|---|---|
| **T1** | **≥0.5 µs** | /READY and /WRTGATE sequencing in the diagram; preserve readiness qualification |
| **T2** | **0–43 µs** | Delay from write gate to the drive's **internal /ERASE** response; **not** an STM32 host pin |
| **T3** | **≥1.8 µs** | From write-gate assertion to first effective WR transition |
| **T4** | **≥2 µs** | From last WR transition to write-gate release |
| **T5** | **≥0.5 µs** | /READY relation at end of the write window |
| **T7** | **480–590 µs** | Extension of the **internal erase** operation after gate release; **not** proof that host writes continue for this duration |

The graphical timing boundaries must still be tested at the target connector. These are constraints of the original interface, **not software delays the emulator should add to simulate a physical erase head**. T2/T7 explain how a real floppy preserves an erased splice; their consequence for the emulator is to handle the affected **bit regions** correctly, not to simulate erasing magnetic oxide for half a millisecond.

**[DECISION]** Treat `/WRREQ` low as the **write-gate-active** condition, qualified by selected/active drive and current media/write permissions. `/WRREQ` high ends the permitted capture window. The STM32 must timestamp **both gate edges and all WR edges** using a common timebase.

**[OPEN]** Verify the Macintosh Plus IWM's behavior during normal completion vs **write underrun**: Apple's IWM description says WRREQ may be disabled on underrun. Do **not** assume that a rising gate alone proves a successful normal sector write; correlate length, field structure and any observed abort indicators where available.

**[DECISION]** A host write on a protected or absent medium must never reach flash. Host write-protect status alone is not sufficient: the firmware must enforce it even if the ROM cached an earlier writable state ([Part 1 / §9.8](06-part-1-drive-interface.md#98-write-protect-sources-caching-and-enforcement)).

### 12.4 Macintosh IWM write state, byte handshake and underrun

**[IWM]** The original IWM receives the ROM's **pre-encoded eight-bit GCR codewords**, loads a write shift register and serializes **MSB first**. The IWM does *not* construct the six-bit GCR lookup codes or synthesize `SYNC6`; those are ROM data. Its write-buffer handshake is **processor-internal**, not a separate byte-handshake bus at the Sony connector.

The ROM follows this concrete register path:

| ROM action | IWM register/state effect | What the emulator actually observes |
|---|---|---|
| Read `Q6H` in sense state | Set L6 high | Internal IWM state; no new drive data byte |
| First byte written to `Q7H` | L7 becomes high and begins IWM write state | `/WRREQ` gate behavior and subsequent WR edge stream |
| Read `Q6L` while L7 high | Read write-handshake; **bit 7** means write buffer empty | **Not** available as a pin to STM32 |
| Write subsequent codeword to `Q6H` | Feed IWM write buffer | WR transitions emerge at serial bit timing |
| Read `Q7L` at end | L7 low, leave write state | Write gate releases, with finite settling/trailing bits |

**[ROM]** [`Sony_WrData`, lines 2555–2596 and 2702–2719](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2555-L2719) implements this path; [`SONY_DOFORMAT`, lines 2965–3078](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2965-L3078) uses the same write-state handshake in a formatting loop.

**[IWM]** The IWM/SWIM reference documents asynchronous write-buffer loading and **underrun** behavior: the handshake status **bit 6** clears on underrun, while bit 7 indicates whether the IWM can accept another byte. On underrun a continued good data stream is **not** assured. [Apple *SWIM Chip User's Reference*, pp. 10–12](https://bitsavers.org/pdf/apple/disk/sony/SWIM_Chip_Users_Ref_198801.pdf); [Apple SWIM/IWM design specification, 343S0061-A, sheet 52](https://www.bitsavers.org/pdf/apple/disk/sony/SWIM_343S0061-A_1988.pdf). The STM32 cannot read these handshake bits directly; it sees **gate and transitions only**. Do not mistake bit 7 of IWM's handshake register for bit 7 of a byte transmitted on WR.

### 12.5 The ordinary sector-write ROM path

**[ROM]** The Plus is **not a whole-track write-back cache**. A normal write request selects the sector(s) to update, reads address fields until it finds one of them, then performs `E_Sony_WrData` on the matched sector. The ROM's track cache serves reads and **invalidates** written sectors; it does not accumulate dirty encoded tracks to be flushed on later track changes.

~~~text
Select logical track/head
       ↓
Read RD pulses → IWM bytes
       ↓
ROM Sony_RdAddr searches for D5 AA 96
       ↓
ROM decodes track / side / sector / format / address checksum
       ↓
Requested sector?
    no → continue next address field
    yes → prepare 12 tags + 512 data bytes
       ↓
ROM Sony_WrData produces data sync + data-field GCR bytes
       ↓
IWM asserts write gate and serializes WR level transitions
       ↓
STM32 captures WR transitions against known virtual track phase
       ↓
Stop gate; validate/merge changed field and schedule persistence
~~~

**[ROM]** The decisive call occurs at [`Sony_ReSeek`, lines 888–1012](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L888-L1012): a matching **address field is read first**, then `E_Sony_WrData` is invoked. **No address-field rewrite** takes place in that ordinary data write. [Cache invalidation, lines 702–715](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L702-L715).

**[DECISION]** The STM32's read stream and its monotonically advancing **virtual bit position** provide information about which sector the Macintosh just saw. A validated address/sector position map can link the **write gate** with a likely target sector **without 6-and-2 decoding the write payload**. This is a design advantage, **not permission to assume that write gate is aligned to the start of the data field**.

### 12.6 Exact normal sector data-field stream

**[ROM]** For a normal data-field overwrite, `Sony_WrData` emits the following codeword/bit-cell sequence. Refer to [`SONY_WRDATAMKS`, lines 2531–2539](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2531-L2539), [`Sony_WrData`, lines 2547–2719](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2547-L2719), and the fuller [read-side format reference, Part 2 / §10.6](06-part-2-serial-gcr-read.md#106-full-sector-structure--exact-macintosh-plus-freshly-formatted-reference):

~~~text
EXISTING ADDRESS FIELD IS NOT REWRITTEN:
    [address SYNC6] D5 AA 96 [T] [S] [H] [F] [X] DE AA
    [nominal FF pad, potential physical splice]

NORMAL ROM WRITE STREAM:
    SYNC6                              6 transmitted bytes / 48 bit cells
    D5 AA AD                           3-byte data-field mark
    [S]                                1 encoded sector identifier byte
    [699 GCR codewords]                12 tags + 512 data, 6-and-2 encoded
    [4 GCR checksum codewords]         three 8-bit running checksums encoded
    DE AA                              data end markers
    FF FF                              trailing write-off padding
~~~

**Nominal transmitted size** (not including possible start-up pipeline/partial off-pad behavior):

~~~text
    6 + 3 + 1 + 699 + 4 + 2 + 2 = 717 transmitted bytes
    717 × 8 = 5,736 bit cells
    5,736 / 489,600 = 11.715686 milliseconds at project nominal rate
~~~

**Note:** Exactly **699 data codewords plus four checksum codewords = 703** are encoded following the **separate sector-ID nibble**. Do not count the eight-bit encoded sector ID among the 699 payload codewords. Every field and SYNC6 group is sent as serialized bits; the in-flash packed-byte alignment can differ from these ROM codeword boundaries.

**[OPEN]** The source-level count establishes the *intended transmitted codewords*. The actual **WR gate length** and the first/last recorded bit cells can include IWM pipeline/precharge/trailing effects. Use this count as a structural validation reference, **not as a fixed 5,736-bit gate-high/low duration or unconditional byte offset**.

### 12.7 Recovering bits and phase from WR edges

**[DECISION]** Timestamp **every WR rising and falling edge** during qualified gate time with a high-resolution STM32 timer/input-capture facility or equivalent deterministic logic. Never sample a fixed WR logic level once per bit cell and call it encoded data; **the transitions encode ones**. Maintain a capture-ring buffer that cannot be blocked by SPI flash erases or foreground file-management work.

A **proposed decoder** operates in two stages:

1. **Acquisition:** Capture `(timestamp, edge polarity)` and the two gate edges; filter only demonstrable electrical glitches using a specified threshold, not an arbitrary software debouncer. Capture samples must remain ordered and overflow detectable.
2. **Reconstruction:** Estimate transition spacing as integer multiples of the global/observed bit-cell period `Tbit`. Consecutive transitions one cell apart encode `11`, two apart `101`, and three apart `1001`. Use `SYNC6` repetitions and/or field prologues to establish the **phase and byte synchronization**, especially at the beginning of a write burst. Insert zeros only for **missing bit cells** between timed transitions. Track the exact reconstructed valid bit count and any undecodable runs.

~~~text
Given known phase and successive transition times ti, ti+1:
    cells = nearest_valid_integer((ti+1 - ti) / Tbit)
    cells = 1 → next bit is 1; no intervening zero
    cells = 2 → 0 then 1
    cells = 3 → 0, 0 then 1
    otherwise → anomaly / special-case investigation (not silent repair)

Do NOT infer initial zero cells before the first edge,
or final zero cells after the last edge, from the gate boundaries alone.
~~~

The **first edge's phase relative to write-gate assertion is not guaranteed to be a whole bit-cell boundary**. The source includes explicit first/last transition-to-gate guards. Recover frame boundaries with the known serialized `SYNC6` / mark stream and retain gate timing. Sync detection must be **bit-aware**, because `FF 3F CF F3 FC FF` is **not** six independent, byte-aligned GCR `FF` codewords.

**[PROPOSED]** Maintain both a compact reconstructed packed-bit buffer and a small amount of **edge timing diagnostics** (for example outlier count, measured bit-cell period and start/end phase). A full waveform archive is not required in the final SPI cartridge for ordinary standard tracks, but edge timestamps are extremely helpful during bring-up and anomaly diagnosis.

**[OPEN]** Quantify input jitter, timer resolution, clock crossing, allowable interval error and comparator/level-shifter delay. The 489,600 bit/s nominal rate is settled as project timing baseline, while **physical measurement remains pending**.

### 12.8 Identifying the target sector from virtual read position

**[DECISION]** The STM32 knows the selected **radial track** (digital STEP state), **side** (SEL), **bitstream phase** (its continuous cyclic RD scheduler), and a validated **sector-position map** derived when the raw-GCR track is loaded. It can therefore associate a normal gated write with the **address field most recently made visible** to the Macintosh.

**[PROPOSED]** For each standard track, map the bit positions of address prologue `D5 AA 96`, encoded address values, address end `DE AA`, data sync, data prologue `D5 AA AD`, data-field end and safe adjacent gap bounds. The firmware records the **read phase at gate assertion**, as well as actual timestamps. The candidate target should be the most recently **completed valid address field**, not merely the “nearest sector number” in a static array.

**[OPEN — important]** The ROM's recognition of `D5 AA 96` takes finite time **after** the STM32 has emitted each bit, and the IWM's read-data deserializer is not exposed on the connector. The time from the last emitted address bit until gate assertion depends on the ROM's execution path. Therefore, this mapping is **a candidate checked by bounded timing and structural validation**, not an absolute identity proof just from a gate transition. On ambiguous, missing/corrupt address fields or nonstandard tracks, do not silently overwrite the wrong sector. Further validation may inspect the **encoded sector-ID nibble** in the captured data stream and compare it with the target's header, **without full payload 6-and-2 decoding**.

### 12.9 Read-to-write handoff and write splice

The host stops using the data read stream to recognize the address, then enters IWM write state. **Real hardware has no index signal** telling the Macintosh where it is within the revolution. The existing address field provides the reference. In the emulator, this translates to a bit position in the selected track and a **time-stamped write gate**.

**[PROPOSED]** Retain the virtual bit clock while WR is active (even if the RD output is suppressed or the host has selected a different RD source). Use a common high-resolution timebase to relate:

~~~text
tA: last valid address-field end transmitted on RD
tG: /WRREQ falling (write begins)
t0: first legal WR transition
ti: later WR transitions
tL: last legal WR transition
tE: /WRREQ rising (write ends)

A candidate write position follows from the continuously advancing
virtual track phase, then is checked against the data-sync/mark pattern,
nominal field span and safe boundaries of the next address field.
~~~

**[DECISION]** The update must preserve the **other data fields and address fields**; sector GCR payloads need not be decoded and re-encoded. **Do not** treat `tG` as the start of a byte-aligned `SYNC6` or replace a fixed number of flash *bytes* from that instant. The effective write splice may begin inside sync/padding and end in trailing `FF` material.

**[PROPOSED — standard tracks]** Use the verified start of `D5 AA AD` and surrounding mapped synchronization region as **anchors**. Merge captured serialized bits within a bounded write region that can absorb phase/length variation in **known padding/synchronization**, preserving the next sector's address mark and all unaffected sectors. A canonical SYNC6 generator (Part 3) may regenerate only validated standard sync regions **after** the merge and **without silently shifting the rest of the track**.

**[OPEN]** Decide whether to perform (a) literal timestamp-to-track bit-cell splicing, (b) structural data-field replacement with validated sync rebuild, or (c) a hybrid. These have different consequences for copy-protection and original timing fidelity. A selected algorithm needs both simulator fixtures and a real IWM loopback test; a normal ROM write must work even if its emitted timing differs slightly from the emulator's read clock.

### 12.10 Whole-track formatting and first-use blank media

**[ROM]** Macintosh formatting uses `Sony_CtlFormat` → `SONY_FORMATTRACK` → `SONY_DOFORMAT`, **not** the ordinary data-only writer. In one gated stream it sends **200 `SYNC6` startup groups**, then for every sector: `GapSync-1` additional sync groups, the **27-byte address/data prologue template**, **703 × `$96`** freshly formatted data/checksum nibble bytes and **`DE AA FF FF`**. The template itself contains one additional address `SYNC6`. The formatter can rewrite a track with a modified `GapSync` after its readback check.

**[ROM]** [`SONY_DOFORMAT` lines 2958–3078](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2958-L3078); [`SONY_FORMATTRACK` lines 2883–2955](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2883-L2955); full format stream in [Part 2 / §10.6](06-part-2-serial-gcr-read.md#106-full-sector-structure--exact-macintosh-plus-freshly-formatted-reference).

**[DECISION]** Formatting cannot rely on a pre-existing valid sector map. A **blank but present/writable** SPI cartridge must remain recognizable as present/ready enough for the Plus to initiate formatting, even if RD cannot return meaningful `D5 AA 96` sector fields yet. The emulator can use the standard 80-track geometry/profile to provision an initial logical track descriptor with a **nominal sector count** and sufficient unformatted-track storage. The firmware need not model a physical speed zone.

**[PROPOSED]** Classify **candidate format** from a sustained gate with repeated SYNC6 and valid address marks across multiple sectors; do not classify it **only** by a long gate duration. Build/replace the selected logical track's complete cyclic bitstream, recording its valid length, sector map, side, format identifier and integrity. The formatter's **200 initial SYNC6 groups are transient write lead-in, not a permanent 200-group prefix**: the captured stream may exceed the selected virtual revolution, requiring correct wrap/overwrite semantics or a validated canonical construction.

**[OPEN — critical bootstrapping issue]** The ROM **reads back sector 0 immediately after formatting** as its fit test. The STM32 must have prepared a coherent circular track, and the outgoing RD scheduler must expose it, *before* that readback. The precise `GapSync` fitting model, initial revolution length/phase for blank media, and readiness handoff must be designed. Format retries that rewrite the same track must not produce duplicate sector maps or unbounded growth.

**[DECISION]** The first successful Macintosh format also supplies a long **known WR timing pattern** for the **one global SYNC6 calibration**. A later image import may replace the raw-GCR data while preserving calibration and cartridge identity.

### 12.11 SYNC6 automatic calibration during formatting

**[DECISION]** One **global** GCR bit-cell timing record is learned from the Macintosh's format-time SYNC6 stream and stored in the cartridge's **global metadata**; do not place a separate calibration in each track descriptor or maintain five different GCR clocks.

~~~text
ROM-format preamble:
    200 × (FF 3F CF F3 FC FF)
    = 200 × 48 = 9,600 bit cells
    = 8,000 known transition bits
    ≈ 19.608 ms at nominal 489,600 bit/s

Valid intervals in repeated SYNC6:
    one bit-cell:  Tbit
    three cells:   3 × Tbit
~~~

**[PROPOSED]** During the first long format preamble, detect the known pattern at edge-interval level, accumulate timer ticks across many **complete 48-bit groups**, reject edge glitches/missing transitions, then estimate `Tbit` using a robust statistical method. Save measurement uncertainty and a validity/integrity marker. Finish an ongoing capture with one stable timer calibration; apply changes to the **RD clock only at a safe boundary**, never mid-field.

**[OPEN]** Select permitted ppm/jitter bounds and success/fallback rules, and determine how to handle calibration recorded by a different Macintosh or a different STM32 oscillator. SYNC6 does **not** calibrate RD output pulse width, pin voltage, flash write completion or TACH—these are independent host-visible characteristics. Initial nominal timing must work well enough to allow the first format.

### 12.12 RAM buffers, write acceptance and flash persistence

**[DECISION]** Flash is **not written directly per incoming WR edge**. Decouple deterministic write capture from the erase/program behavior of SPI NOR. Keep the current track's packed raw-GCR stream available for readback and ordinary sector modifications; maintain a temporary WR edge/packed-bit capture buffer. Formatting requires capacity for a continuous whole-track write operation; it cannot be limited to a single 717-byte sector.

**[PROPOSED]** Use a transaction pipeline:

~~~text
Qualified gate opens
    → reserve capture transaction (track/side, phase, host state)
    → timestamp/collect all WR edges without flash blocking
Gate closes (or an abort is detected)
    → reconstruct bits and validate signal timing
    → classify NORMAL DATA FIELD / FORMAT / invalid / unsupported
    → validate target, markers, checksum structure as appropriate
    → atomically publish new RAM track version
    → queue persistent flash operation and metadata generation update
    → make readback see the published data, not stale flash bytes
    → commit/recover per approved durable media policy
~~~

**[DECISION]** Ordinary-sector writes may update only a small region but a flash erase may cover much more. Preserve unaffected fields and neighboring flash data and keep the flash allocation **offset/length** descriptors coherent. Do **not** change a track descriptor's valid-bit length silently when rewriting one field. An entire-format overwrite may establish a new track bit length and map as a distinct transaction.

**[DECISION]** **Track-change-only flush is insufficient**: the Macintosh can finish a write and then turn the computer off/eject without changing track. The emulator needs a documented and testable **durability and safe-eject policy**. The ROM's apparent write completion **does not prove** that the cartridge's flash update has finished. Unknown power-fail or incomplete writes must not be reported as clean after restart.

**[OPEN]** Precise crash consistency and safe eject behavior need a concrete SPI-NOR design: erased-sector granularity, page-program times, spare blocks, immutable track allocation vs in-place erase, two directory generations or journal, metadata CRC, write endurance, low-voltage protection, and hot-removal interlock. These are **not settled** by this write-interface specification. A write-capable image must not be marked safely ejectable while acknowledged changes are volatile unless a defined safe-write mechanism protects them.

### 12.13 A proposed STM32 write state machine

The following is a **conceptual design**, not compiled code or an approved peripheral allocation:

~~~text
IDLE / READING
    Keep current track/side, sector map and virtual bit phase.
    If /WRREQ falls:
        capture t_gate and current bit phase
        latch media identity, track/side, writable/ready status
        if not qualified: mark denied; do not modify stored media
        else: arm BOTH-edge WR capture with DMA / timer hardware
        enter CAPTURING

CAPTURING
    Record WR rising AND falling timestamps in bounded capture FIFO.
    Track gate still low; detect overflow, too-long burst, selection fault.
    Do not block on SPI NOR or per-edge GCR arithmetic.
    On gate rise: capture t_end; stop accepting new edges; enter ANALYZE

ANALYZE
    Check qualification remained valid and no overflow/abort.
    Reconstruct bit-cell stream from timestamp intervals + sync anchors.
    If standard normal data write:
        verify sector association and safe splice boundaries
        prepare modified whole-track RAM image
    Else if whole-track format:
        validate address/data structure, map and cyclic length
        prepare a fresh track image; update calibration candidate
    Else:
        record unsupported/invalid transaction and do not corrupt media
    Publish coherent RAM track only after validation.
    Enter PERSIST_PENDING or return to READING as policy permits.

PERSIST_PENDING
    Queue flash-safe changes; never block input edge capture.
    Continue RD from authoritative RAM track snapshot if allowed.
    On step/eject/power-loss, follow a defined durability policy.
    Only advertise a clean/persisted cartridge when confirmed committed.

FAULT
    Count/report capture overflow, timing error, invalid gate,
    unexpected head/track change, write-protection fault,
    invalid sector map, bad format and flash/media failures.
    Preserve previous valid raw track/metadata until recovery decisions.
~~~

**[OPEN]** Exact RD gating, IRQ/DMA races, timer overflow handling, flash-write scheduling and the policy for rejecting writes on a drive interface with **no explicit per-sector acknowledgement line** require hardware validation. Do not invent a drive-to-host write-success signal; the ROM relies on IWM behavior and subsequent readback/errors.

### 12.14 Timing and buffer budget

**Numbers below are nominal **engineering calculations**, not proof of worst-case STM32 performance.**

| Quantity | Derivation | Initial figure |
|---|---|---:|
| Global GCR bit-cell time | `1 / 489,600` | **2.042484 µs** |
| Normal GCR transition intervals | 1/2/3 × `Tbit` | **2.0425 / 4.0850 / 6.1275 µs** |
| One encoded GCR byte | 8 × `Tbit` | **16.33987 µs** |
| Data field transmitted by `Sony_WrData` | 717 × 8 cells | **5,736 cells; 11.7157 ms** |
| Formatting lead-in | 200 × 48 cells | **9,600 cells; 19.6078 ms** |
| Initial format sector entry at `GapSync=7` | (`6×7+728`) × 8 | **6,160 cells; 12.5817 ms** |
| Maximum ordinary WR edge rate | One transition per cell | **489,600 edges/s** |
| Packed raw bitstream transfer rate | 489,600/8 | **61,200 bytes/s** |
| Four-byte raw timestamps at max transition rate | 4 × 489,600 | **1,958,400 bytes/s** |

A 717-byte **packed sector-capture buffer** is only a **payload footprint guide**; allow for gate/startup phase, full bit count, timestamps, spare padding and malformed/extra transitions. A formatted whole track requires **far more** than 717 bytes, and **800K standard format length varies by logical track**. Timer/DMA bandwidth, ring-buffer depth and flash worst-case stalls must be validated rather than chosen from averages.

**[OPEN]** If all edge timestamps are stored for a format burst, they can exceed one packed-track buffer by several times. Design a bounded streaming decoder or suitably sized DMA/ring buffer while maintaining full both-edge capture and avoiding overruns.

### 12.15 Verification/acceptance matrix

| Test | Expected evidence/pass condition |
|---|---|
| **WR electrical capture** | Both rising and falling WRTDATA transitions detected; no confusing them with RD pulses |
| **Gate polarity** | `/WRREQ` low enables qualified recording, high ends it; disabled or protected medium never modified |
| **Sony T3/T4 guards** | First edge and last edge placed/recognized relative to gate per §3.4.5.1; no invented external /ERASE |
| **IWM asynchronous handshake** | Correct ROM 6-and-2 encoded serial stream; host sees no spurious buffer underrun due to emulator behavior |
| **First normal sector write** | Correct address field remains unchanged; only selected sector data region changes; readback succeeds |
| **Other sectors preserved** | Before/after raw bits and address marks outside the accepted splice remain intact |
| **Both sides** | Head/side selection and the active write target remain consistent under normal writes |
| **Different normal sectors** | Several sectors updated on the same track without requiring a track change or full track rewrite by ROM |
| **Write phase offset** | Delayed gate/start edge doesn't shift following sector data or corrupt its address mark |
| **Gap / sync behavior** | Field SYNC6, address `DE AA FF`, data marks, ending `DE AA FF FF` meet standard layout |
| **Whole-track format** | Recognize extended write, build one valid cyclic track, correct sector map and 2:1 interleave |
| **Format retry** | Reformatting same track with adjusted `GapSync` replaces rather than appends data |
| **Immediate post-format readback** | ROM finds sector 0 and validates zero data and checksums with no false delayed readiness |
| **SYNC6 timing calibration** | One validated *global* timing record from many complete groups; later image import preserves it |
| **Track metadata** | Sector count, valid bits, flash offset/length and checksum stay consistent on every update |
| **Missing WR transitions** | Detect invalid intervals/underrun-like truncation; do not silently invent a valid sector |
| **Capture overload** | Buffer overflow reported deterministically; no accepted corrupt data committed |
| **Write protection / removal** | No flash modification after protection/removal; no cross-cartridge stale-buffer write |
| **Power loss during flash work** | Previous or new committed directory/data generation recoverable; no false-clean stale metadata |
| **Eject with dirty cache** | Media release policy ensures no acknowledged changes disappear |
| **Throughput under load** | WR both-edge capture stays lossless while RD/TACH/status tasks and SPI operations compete |
| **Bench loopback** | Compare WR/gate, reconstructed bits, resulting RD and ROM-visible sector data against target Plus |

**Recommended test progression:** (1) electrical both-edge/gate fixture; (2) known SYNC6 and arbitrary bit-pattern capture; (3) ordinary `Sony_WrData` simulation at variable gate phase; (4) real Plus one-sector write/readback; (5) whole-track format and immediate ROM sector-0 read; (6) four zone-boundary/side changes with a *single* GCR clock; (7) flash power-cut/eject; (8) external image conversion and boot.

### 12.16 Decisions and remaining engineering gates

**Established:**

- **[SONY]** WR carries **level transitions on both edges**, each one a flux reversal; RD carries **pulses**. Qualify writes by active drive, active-low gate and physical-media write permission.
- **[ROM]** Ordinary writes follow **read-address-then-write-data**, with data-only replacement; formatting is **extended whole-track** streaming. ROM generates `SYNC6`, GCR codewords and checksums; IWM serializes and handshakes.
- **[DECISION]** STM32 captures WR into RAM, retains its continuous virtual track position, uses track/side/sector metadata and preserves unaffected raw-GCR bit regions. It does **not** 6-and-2 decode/re-encode user data in the normal live path.
- **[DECISION]** One cartridge/global SYNC6-calibrated bit-cell rate for all logical tracks; *sector count* supplies standard TACH frequency, independent of GCR timing.
- **[DECISION]** SPI-flash storage is bit-packed raw GCR; globally extensible and per-track offset/length metadata; firmware must not silently modify nonstandard tracks.
- **[DECISION]** A completed host write must be handled by a durable storage/eject policy; changing tracks is not a sufficient sole commit trigger.

**Proposed, not yet hardware/ABI approved:**

- Both-edge hardware capture timestamps via timer + DMA/ring buffer.
- SYNC6/field-marker anchored bit reconstruction and verified mapping back to the last address field delivered on RD.
- Whole-track RAM cache with a separate sector-write capture and an extended formatting capture mode.
- Flash transaction/journal or alternate recoverable directory generations, to be defined by the actual SPI NOR erase/program constraints.

**Must still be resolved:**

1. **Pin-verified WR and /WRREQ waveforms** on the specific Macintosh Plus/IWM, including gate assertion/release, byte-pipeline startup, and underrun.
2. Quantified **WR edge interval acceptance/jitter**, phase alignment to the outgoing RD read stream and correct treatment of the first/last partial bit cells.
3. The **write-splice algorithm**, including safe sync/gap replacement bounds, preservation of next-sector headers and authentic versus canonical raw-track modes.
4. **Blank-media and whole-track-format bootstrap**: sector count/profile, virtual track length, 200 lead-in overwriting, `GapSync` fit, and immediate ROM readback.
5. Specific **STM32 capture peripherals and RAM budget** under full load and potential DMA contention.
6. Precise **SPI flash commit/crash consistency**, sector erase constraints, metadata versioning and safe ejection, with proof at the 2 MiB size target.
7. Robust **external programmer/image-conversion support**, tag handling, lossless import limits and calibration portability.
8. Physical prototype verification against **Apple 699-0452-A §§3.2.5–3.2.6 and §3.4.5** and ROM behavior, with original 20-pin Sony hardware/wiring as reference.

**Document history**

- **2026-11-09:** Created Part 4 / Section 12. Separated **both-edge, level-transition WR** from **pulse RD**, captured Sony gate timing including internal erase, traced ROM ordinary sector writes and track formatting, set a proposed STM32 capture/merge pipeline and validation gates while leaving bit-splice, flash durability and hardware waveforms explicitly open.
