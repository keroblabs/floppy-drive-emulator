# Sony MFD-51W Drive Interface Protocol — Part 4: Serial GCR Writing

**Document:** 06 — Part 4  
**Protocol section:** **12 — Serial GCR write-data capture, format, track replacement and persistence**  
**Status:** ROM/source-grounded electrical protocol and approved structured-GCR write/cache architecture; specific DMA/flash/sector-record binary ABI, gate/splice timing and durability recovery require prototype verification  
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

**[DECISION]** STM32 tracks the selected radial track/side, virtual RD bit phase and the sector-position map of its **fully reconstructed cyclic RAM track**. The ROM reads the target **address field** before issuing an ordinary gated **data-field-only** write. The STM32 may associate the incoming write with the most recent *complete* valid address field, then check the separately encoded sector ID in the incoming data header. This must be a **validated association**, not a guess based only on gate assertion.

**[DECISION]** A standard flash record preserves the **10-byte encoded address field**, **4-byte data header**, and **703 encoded payload/checksum codewords**. An ordinary write replaces only the latter two components (after validation); **the existing address field remains unchanged**, and other sectors remain untouched. The full 735-byte in-RAM record and generated sync/gap region determine the rebuilt playback track.

**[OPEN]** The ROM/IWM's delay between address recognition and gate assertion, WR start/end pipeline and sector ID decoding must be bench-verified. Do not silently change the wrong record on a missing/ambiguous address, bad identifier or invalid nonstandard sector map.


### 12.9 Read-to-write handoff and write splice

**[DECISION]** Even in standard structured mode the hardware must capture the **real WR timing**, both transition polarities and active-low gate and validate a correctly framed GCR field. The gate edge is **not** an encoded-byte boundary. Preserve the shared high-resolution virtual-track/WR clock and the start/end timing diagnostics.

~~~text
tA: last completed valid address field emitted on RD
tG: /WRREQ falling edge
t0: first reconstructed WR transition
ti: further edges
tL: last WR transition
tE: /WRREQ rising edge
~~~

**[DECISION — structured standard tracks]** After reconstructing the incoming serial GCR codewords, identify `SYNC6`, `D5 AA AD`, the encoded sector ID, **703 GCR data/checksum bytes** and `DE AA FF FF`. Validate the header/target against the previously presented stored address. Copy the **data header + 703 codewords** into the selected in-RAM record, rebuild/correct only the standard field's known generated sync/padding/gap, then atomically publish the updated RAM track. **No exact physical flux-splice recording is required** for standard tracks; preserve every other sector record and the complete existing 10-byte address header.

**[OPEN — RAW_TRACK]** For untouched packed-bit nonstandard tracks, a literal bit-cell splice or explicitly unsupported-write policy must be designed separately. Do not rewrite copy-protected or unusual tracks into standard layout merely because they contain familiar markers. Hardware proof is needed for first/last WR cells, timing tolerance, truncation and race-free stream publication.


### 12.10 Whole-track formatting and first-use blank media

**[ROM]** `Sony_CtlFormat` → `SONY_FORMATTRACK` → `SONY_DOFORMAT` writes **200 consecutive SYNC6 groups** at the start of a continuous gated burst, then per sector `GapSync−1` sync groups, a 27-byte address/data template, **703 encoded zero-data/checksum codewords** and `DE AA FF FF`. Afterward it **immediately reads sector 0** and may repeat formatting with a smaller `GapSync`. See [ROM format source](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2883-L3078).

**[DECISION — ROM-grounded early discriminator]** The ordinary `Sony_WrData` path begins with **one SYNC6**; the format path begins with **200**. Detecting **at least 50 consecutive complete SYNC6 groups inside one qualified write gate** classifies a **candidate whole-track format** (≈4.902 ms at the nominal 489,600-bit/s rate). This is a trigger for early buffer/flash preparation, **not proof of success**. Maintain full capture and confirm the address fields, interleave, expected sector count, encoded payload/checksums and completion.

**[DECISION]** On a healthy blank cartridge, advertise sufficient present/ready state to allow Macintosh formatting. The standard profile supplies preliminary track/side and expected sector-count metadata, without a motor-zone emulator. On recognizing candidate formatting, prepare an incoming RAM track buffer and **begin erase of the isolated track allocation or safe spare erase units as permitted**; never erase other tracks or shared metadata. Physical formatting is destructive and an interrupted format need not preserve the old track, but a failed write must not corrupt the directory or unrelated tracks. A recoverable flash transaction scheme is still [OPEN].

**[DECISION]** For **STRUCTURED_GCR**, keep the sector's **complete 10-byte encoded address field + 4-byte data header + 703 GCR codewords** (735-byte record including 18 bytes of further metadata). **Do not persist** the 200-group preamble, per-sector SYNC6/gap, standard `FF` pad or `DE AA FF FF`. Use the ROM-derived physical sector order and accepted `GapSync` to rebuild the circular RAM track and its known bit length, making it **available for immediate ROM sector-0 readback** regardless of pending SPI persistence. Adjust/rebuild on ROM format retry rather than appending duplicate sectors or retaining a stale track map.

**[OPEN — bootstrapping]** Model/verify the exact wrap seam and amount of residual sync after 200-group overrun, gap-fit/sector-0 return, first-use provisional timing, and behavior if interrupted after partially formatting a valid track. Gate release alone is not sufficient to declare format success; validate before publishing a complete readable track.

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

**[DECISION]** **At least three independently managed complete-track RAM buffers of at least 12 KiB each (>=36 KiB)**, plus separate DMA/ring write-edge capture buffers, page staging and cache descriptors. A buffer may be ACTIVE/READABLE, INCOMING/RECONSTRUCTING or DIRTY/FLUSHING, with roles reassigned dynamically; pin a buffer/generation when in active RD use or until its sole dirty copy is safely committed. No incoming WR edge or active RD pulse may block on SPI NOR erase/page-program.

**[DECISION]** Standard persistent representation is **735 bytes per sector**: **32-byte metadata containing the preserved complete 10-byte address header and 4-byte data header**, plus **703 original encoded data/checksum codewords**. The exact byte offsets within the proposed 32-byte metadata (e.g. 10 address + 4 data header + 1 physical order + 1 flags + 4 CRC32 + 12 reserved) are not yet a frozen ABI. Preserve data checksums as the last four of the 703 codewords; do not routinely regenerate them.

**[DECISION]** The current published RAM track is the authoritative source for immediate read-after-write/format-readback. The flash worker independently persists changed sector records, track descriptors and calibration according to a **safe commit/eject policy**. Coalesce multiple dirty sector writes on the same track where possible. Track-change-only flush is **insufficient**; the Macintosh may power down or eject after a write without changing track.

**[DECISION]** Standard track records have **dedicated 4-KiB erase-aligned allocations**: 12-sector tracks require 3 erase units; 8–11-sector tracks require 2. Page program commands are 256-byte pages. No erase may destroy neighboring track records or centralized metadata. The entire 800K disk uses **1,441,792 bytes (1,408 KiB)** of such allocations, leaving 640 KiB of the 2-MiB cartridge before directory/journal/spares. RAW_TRACK has an alternate track-level packed-bit layout; it must not be normalized.

**[DECISION]** The SPI worker may begin erasing safe target units when a format is recognized and may program/verify a completed buffered track while the ROM is formatting a subsequent track. Immediate readback **must not wait for SPI completion**. Streaming stable pages during reception is a future optional optimization, **not required** by the first implementation. Avoid committing unvalidated partial sector/track data.

**[OPEN]** Specific chip/SPI transfer and polling overhead, transaction atomicity, wear, dirty-generation eviction, power-loss recovery and safe physical cartridge release. A three-buffer minimum does not prove arbitrary write bursts can be absorbed indefinitely; implement deterministic overflow/backlog handling without pretending the interface has a host pause/acknowledgment mechanism.


### 12.13 A proposed STM32 write state machine

**Conceptual design; exact peripheral allocation and flash transaction ABI remain open.**

~~~text
READY / READING
    Provide prepared cyclic RAM GCR track and correct RD/TACH/status.
    On qualified /WRREQ falling:
        latch cartridge/track/side/RD phase and selected clean generation
        arm independent BOTH-edge DMA timer capture
        begin stream-based GCR/format discrimination

CAPTURE / CLASSIFY
    Process bounded capture ring without flash or filesystem stalls.
    If one SYNC6 followed by D5 AA AD: candidate ordinary data-field write.
    If >=50 consecutive SYNC6 within same gate: candidate format;
         reserve incoming whole-track RAM buffer, schedule safe early erase.
    Detect missing/invalid cells, wrong identifiers, overflow, protection.
    At gate release finish framing; do not assume success from gate alone.

VALIDATE / BUILD
    Ordinary: validate one encoded header, 703 data/checksum codewords,
         target vs last valid address, generated field boundaries;
         replace data header + codewords in existing structured record.
    Format: validate sector array, stored address/data headers, order/count,
         accepted gap and seam; reconstruct whole cyclic RAM track.
    RAW_TRACK: preserve exact raw representation or explicit unsupported case.
    Publish a coherent RAM generation for immediate readback only if valid.

ASYNC FLASH WORKER
    Independently erase safe isolated extents, program verified records,
    update track metadata and recoverable directory/transaction state.
    Coalesce repeated dirty updates; retain pinned dirty generations.
    Never block RD or WR capture; preserve safe eject and recovery state.

FAULT / BACKLOG
    Track gate/underrun, edge decode, overflow, no spare buffer,
    flash busy/failure, protection, removal and interrupted formatting.
    Do not silently discard accepted sectors or corrupt other tracks.
~~~

**[OPEN]** Hardware timing, cache pressure, metadata atomicity, flash durability and ROM-compatible circular format check require bench fixtures.


### 12.14 Timing, SRAM and flash throughput budget

**Reference engineering numbers** use the agreed nominal **489,600 bit cells/s** (measurement pending) and Microchip [SST26VF016B maximum operation timings](https://ww1.microchip.com/downloads/en/DeviceDoc/SST26VF016B-2.5V-3.0V-16-Mbit-Serial-Quad-IO-SQI-Flash-Memory-Data-Sheet-DS20005262F.pdf) (**4-KiB erase ≤25 ms**, **256-byte page program ≤1.5 ms**). The actual SPI NOR part is not yet selected. All flash totals below are sequential **internal erase + program only**, excluding data transfer, status polling, verification, metadata, retries and wear-management overhead.

| Quantity | Reference figure |
|---|---:|
| Nominal bit-cell time | 2.042484 µs |
| Ordinary 717-byte serial data-field write | 5,736 cells ≈ 11.716 ms |
| One SYNC6 group | 48 cells ≈ 98.04 µs |
| 50 SYNC6 discriminator | 2,400 cells ≈ 4.902 ms |
| 200 SYNC6 format preamble | 9,600 cells ≈ 19.608 ms |
| Peak WR transition rate | 489,600 transitions/s |
| Incoming bit-packed data equivalent | 61,200 bytes/s |
| 4-byte timestamps at peak edge rate | 1,958,400 bytes/s (ring/streaming required) |
| **RAM complete-track buffers** | **>=3 × 12 KiB = 36 KiB**, plus DMA rings/staging |

Standard sector records: **32 metadata bytes + 699 data GCR codewords + 4 data-checksum GCR codewords = 735 bytes**. Formatting transmits the temporary 200-group preamble and standard sync/gaps, but those generated portions **are not persisted** for STRUCTURED_GCR.

| Sectors/track | Records used | 256-byte pages | 4-KiB erases | Flash internal max (erase + program) | ROM format stream at `GapSync=7` |
|---:|---:|---:|---:|---:|---:|
| 12 | 8,820 B | 35 | 3 | **127.5 ms** | ≈170.6 ms |
| 11 | 8,085 B | 32 | 2 | **98.0 ms** | ≈158.0 ms |
| 10 | 7,350 B | 29 | 2 | **93.5 ms** | ≈145.4 ms |
| 9 | 6,615 B | 26 | 2 | **89.0 ms** | ≈132.8 ms |
| 8 | 5,880 B | 23 | 2 | **84.5 ms** | ≈120.3 ms |

`ROM duration = [200 × 6 + N × (6×GapSync + 728)] / 61,200 seconds.` It excludes immediate sector-0 readback, gap-adjustment retries and inter-track/head changes; **no fixed pause between tracks is guaranteed**. The timing comparison suggests a viable three-buffer pipeline, **not a deterministic guarantee** of unlimited incoming writes. Ordinary repeated sector overwrites may have a more difficult flash erase/commit workload than a complete track format. Test sustained queue occupancy and worst-case chip times with actual firmware and metadata commits.

### 12.15 Verification/acceptance matrix

| Test | Expected evidence/pass condition |
|---|---|
| **WR electrical capture** | Both rising and falling WRTDATA transitions detected; no confusing them with RD pulses |
| **Gate polarity** | `/WRREQ` low enables qualified recording, high ends it; disabled or protected medium never modified |
| **Sony T3/T4 guards** | First edge and last edge placed/recognized relative to gate per §3.4.5.1; no invented external /ERASE |
| **IWM asynchronous handshake** | Correct ROM 6-and-2 encoded serial stream; host sees no spurious buffer underrun due to emulator behavior |
| **First normal sector write** | Correct address field remains unchanged; only selected sector data region changes; readback succeeds |
| **Other sectors preserved** | All non-target encoded sector records and address fields unchanged; generated standard sync/gap may be rebuilt deterministically; RAW_TRACK is replayed verbatim |
| **Both sides** | Head/side selection and the active write target remain consistent under normal writes |
| **Different normal sectors** | Several sectors updated on the same track without requiring a track change or full track rewrite by ROM |
| **Write phase offset** | Vary gate/first-edge phase; structural data-field replacement finds same encoded target without shifting or damaging adjacent headers |
| **Gap / sync behavior** | Field SYNC6, address `DE AA FF`, data marks, ending `DE AA FF FF` meet standard layout |
| **Whole-track format** | Detect >=50 consecutive SYNC6 within one gate; validate all sectors/headers; rebuild one valid cyclic RAM track, correct 2:1 interleave, and preserve exactly 735 bytes per structured sector |
| **Format retry** | Reformatting same track with adjusted `GapSync` replaces rather than appends data |
| **Immediate post-format readback** | ROM finds sector 0 and validates zero data and checksums with no false delayed readiness |
| **SYNC6 timing calibration** | One validated *global* timing record from many complete groups; later image import preserves it |
| **Track metadata** | Both preserved headers + 703 GCR codewords, CRC/flags, interleave, accepted gap, valid bit length and directory remain coherent |
| **Missing WR transitions** | Detect invalid intervals/underrun-like truncation; do not silently invent a valid sector |
| **Capture overload** | Buffer overflow reported deterministically; no accepted corrupt data committed |
| **Write protection / removal** | No flash modification after protection/removal; no cross-cartridge stale-buffer write |
| **Power loss during flash work** | Previous or new committed directory/data generation recoverable; no false-clean stale metadata |
| **Eject with dirty cache** | Media release policy ensures no acknowledged changes disappear |
| **Throughput under load** | Three >=12-KiB buffers + separate DMA rings sustain normal ROM formatting while flash work runs, including worst-case 12-sector erase/program and dirty-cache pressure; no loss or false durability |
| **Bench loopback** | Compare WR/gate, reconstructed bits, resulting RD and ROM-visible sector data against target Plus |

**Recommended test progression:** (1) electrical both-edge/gate fixture; (2) known SYNC6 and arbitrary bit-pattern capture; (3) ordinary `Sony_WrData` simulation at variable gate phase; (4) real Plus one-sector write/readback; (5) whole-track format and immediate ROM sector-0 read; (6) four zone-boundary/side changes with a *single* GCR clock; (7) flash power-cut/eject; (8) external image conversion and boot.


### 12.16 Decisions and remaining engineering gates

**Established:**

- **[SONY]** WR uses both rising/falling **level transitions** for recorded ones; RD emits pulses. Qualify active-low /WRREQ, media selection and protection; do not confuse internal erase-head timing with a host signal.
- **[ROM]** Ordinary `Sony_WrData` starts with one SYNC6 and writes a data field only; format `SONY_DOFORMAT` starts with 200 consecutive SYNC6 then all sectors. The STM32 uses **>=50 consecutive groups** as an *early candidate format* trigger, not final acceptance.
- **[DECISION]** Standard cartridge representation: **32-byte sector metadata retaining complete address/data headers + 703 unchanged encoded data/checksum codewords**; full cyclic standard sync/gaps/fixed pads are regenerated in RAM. Nonstandard RAW_TRACK remains an alternative bit-packed representation.
- **[DECISION]** At least **three independent >=12-KiB complete-track buffers** plus separate capture rings; structured replacement for normal sector writes; async flash worker, cache generation pinning and immediate RAM readback.
- **[DECISION]** Erase-aligned standard extents (three 4-KiB erase units for 12-sector tracks, two for 8–11), versioned/extendible per-track directory and **one global** calibration. 2-MiB is the target flash size; reference worst-case internal flash operation budgets in `12.14.
- **[DECISION]** No artificial mechanical delays; continuous cyclic RD, ROM-compatible electrical timing/TACH, safe eject and durable accepted writes remain required.

**Proposed exact implementation / still to verify:**

- Timer/DMA edge-capture and SYNC6/field framing with bounded jitter and underrun handling.
- Candidate 32-byte sector-field offsets/CRC algorithm (not a frozen binary ABI).
- Read-to-write address phase and source timing; approved gap/seam algorithm and ROM first-format readback.
- Exact flash transaction/journal/atomicity, interrupted formatting/ordinary writes, worst-case backlogs and power-cut recovery.
- RAW_TRACK nonstandard write-splice support (no silent normalization), external importer fidelity and host-to-host calibration transfer.
- STM32 peripheral/SRAM scheduling, 2-MiB allocation including directory generations, spare blocks and wear.

**Validation gates:** original Plus 20-pin hardware and [Apple 699-0452-A](https://mirrors.apple2.org.za/www.bitsavers.org/pdf/apple/disk/sony/MP-F51_spec.pdf) for gate/WR/RD timing; ROM full-format + immediate sector-0 readback, 400K/800K side/zone changes, multiple sector overwrites, sustained formatting with three buffers, interrupted/power-cut operations and external-programmer boot.

**Document history**

- **2026-11-09 (approved write architecture review):** Settled 32+703-byte structured encoded-GCR records with both stored headers, generated SYNC6/gaps/padding, alternate RAW_TRACK, >=3 complete 12-KiB track buffers, 50-SYNC6 early format detection, structural sector replacement, async flash worker and reference erase/program budget. Kept exact ABI, circular seam and crash-safe durability under validation.

- **2026-11-09:** Created Part 4 / Section 12. Separated **both-edge, level-transition WR** from **pulse RD**, captured Sony gate timing including internal erase, traced ROM ordinary sector writes and track formatting, set a proposed STM32 capture/merge pipeline and validation gates while leaving bit-splice, flash durability and hardware waveforms explicitly open.
