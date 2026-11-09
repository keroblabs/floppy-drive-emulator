# Sony MFD-51W Drive Interface Protocol — Part 5: Formatting, Geometry and 400K Compatibility

**Document:** 06 — Part 5  
**Protocol section:** **13 — Operation: format, sector geometry, sides and 400K compatibility**  
**Status:** ROM-grounded specification with agreed standard-track reconstruction architecture; physical implementation acceptance still requires measurement  
**Target:** Macintosh Plus M0001AP, original Plus ROM `.Sony` driver, IWM and host-facing Sony MFD-51W 800K interface  
**Date:** 9 November 2026

> **Position in the series:** [Part 1 / ``1–9](06-part-1-drive-interface.md) defines drive/status/control/eject and TACH; [Part 2 / `10](06-part-2-serial-gcr-read.md) defines the exact serial GCR stream and RD waveform; [Part 3 / `11](06-part-3-media-architecture.md) establishes the 32+703-byte sector records, RAW_TRACK alternative, global calibration and >=3 track buffers; [Part 4 / `12](06-part-4-serial-gcr-write.md) defines electrical WR capture, early format detection and asynchronous flash persistence. This part specifies **the complete standard logical track that those engines must reconstruct**. It does not repeat the electrical write protocol.

**Evidence notation:** **[ROM]** directly established by Plus ROM source; **[DECISION]** agreed emulator requirement; **[REFERENCE]** engineering computation from agreed reference values; **[OPEN]** implementation measurement or error/recovery detail, not a reopened disk-format decision.

## Contents

- [13.1 Scope and settled principle](#131-scope-and-settled-principle)
- [13.2 Standard 400K and 800K geometry](#132-standard-400k-and-800k-geometry)
- [13.3 Side selection and format identification](#133-side-selection-and-format-identification)
- [13.4 Physical order, interleave and logical sector identity](#134-physical-order-interleave-and-logical-sector-identity)
- [13.5 Preserved address/data headers and checksums](#135-preserved-addressdata-headers-and-checksums)
- [13.6 Deterministic standard-track reconstruction](#136-deterministic-standard-track-reconstruction)
- [13.7 Circular buffer, standard gaps, length and playback phase](#137-circular-buffer-standard-gaps-length-and-playback-phase)
- [13.8 Macintosh ROM formatting and post-format verification](#138-macintosh-rom-formatting-and-post-format-verification)
- [13.9 Blank media, reformatting and interrupted operations](#139-blank-media-reformatting-and-interrupted-operations)
- [13.10 400K compatibility and independent full-disk Verify](#1310-400k-compatibility-and-independent-full-disk-verify)
- [13.11 Acceptance matrix and implementation interfaces](#1311-acceptance-matrix-and-implementation-interfaces)
- [13.12 Settled decisions and remaining engineering gates](#1312-settled-decisions-and-remaining-engineering-gates)

---

## 13. Operation: format, sector geometry, sides and 400K compatibility

### 13.1 Scope and settled principle

**[DECISION]** The emulator must reproduce **the normal Macintosh ROM-compatible serial GCR track**, including its sector count, header values, encoded data, physical sector order, correct SYNC6, gaps/pads, cyclic length and timing. **The removable flash stores only the information needed to reconstruct that stream efficiently**; the Macintosh must not see an invented logical disk format.

The STM32 implements a **logical track**, not a mechanical spinning platter, head actuator or simulated rotational delay. A standard cached **circular track starts at the first sector (sector 0)** by definition. This buffer origin is **not** a reset of the independently advancing host-visible RD playback phase. The reconstructed stream repeats continuously with its profile-correct wrap/synchronization. See ``13.6–13.8.

**[DECISION]** Standard geometry, the requirement for the correct circular track duration, and reconstruction of ROM-compatible gaps are **settled architectural requirements**, not additional user-facing configuration choices. Numerical reference values are derived below. Real-IWM waveforms, gate-to-read handoff, and selected STM32 timer accuracy remain **acceptance tests**, not changes to this agreement.

**[ROM]** The unmodified Plus ROM formats through `Sony_CtlFormat → SONY_FORMATTRACK → SONY_DOFORMAT`; creates address values through `SONY_FILLINMARKS`; ordinary `Sony_ReSeek` scans address fields and identifies sectors by number; format checks sector 0 immediately, while `Sony_CtlVerify` is a separate complete-disk verification call.

### 13.2 Standard 400K and 800K geometry

**[ROM]** All standard tracks use **80 radial positions**. Every 16 tracks the nominal sectors per track/side decreases by one, from 12 to 8. The geometry is the same on both heads when two-sided media are formatted.

| Radial track | Logical profile | Sectors/side | Sector IDs | Standard TACH target |
|---|---:|---:|---|---:|
| 0–15 | 0 | 12 | 0–11 | 394 pulses/s |
| 16–31 | 1 | 11 | 0–10 | 429 pulses/s |
| 32–47 | 2 | 10 | 0–9 | 472 pulses/s |
| 48–63 | 3 | 9 | 0–8 | 525 pulses/s |
| 64–79 | 4 | 8 | 0–7 | 590 pulses/s |

Across **one side**, `16 × (12+11+10+9+8) = 800 sectors`. Across two sides, `1,600 sectors`. Each decoded Macintosh sector has **512 user bytes**, plus **12 tag bytes** embedded in the encoded GCR field:

| Standard medium | Formatted sides | User sectors | User bytes |
|---|---:|---:|---:|
| 400K | Side 0 only | 800 | 409,600 |
| 800K | Both sides | 1,600 | 819,200 |

**[DECISION]** The five sector-count classes, their normal TACH outputs, the **single global 489,600-bit/s nominal GCR cell clock** (2.042484 µs/cell; pending logic-analyser confirmation), and standard GCR field rules are independent firmware contracts. TACH is a **status waveform** derived from the selected track's logical profile, not an instruction to model variable spindle mechanics. Do not introduce five GCR bit-cell clocks.

**[REFERENCE]** The original mechanism uses 60 TACH pulses per revolution. Nominal revolution time is therefore approximately `60 / TACH_rate` seconds, while serialized bit capacity is approximately `489,600 × 60 / TACH_rate` bits. The displayed TACH values are rounded reference rates: **do not assert a uniquely exact circular bit count from those rounded numbers alone**. The standard reconstruction profile must produce the correct cyclical duration and format-compatible gaps; bit length is an output of that fixed ROM-compatible profile, not an additional arbitrary knob.

### 13.3 Side selection and format identification

**[ROM]** `Sony_CtlFormat` checks the drive's `Sides` capability. A two-sided mechanism normally formats both sides; `csParam = 1` explicitly requests **one-sided formatting even on a two-sided drive**. A one-sided mechanism formats one side. The ROM starts at track 0, side 0; if using two sides, it formats side 1 of the same radial track before advancing to the next track.

**[ROM]** `SONY_FILLINMARKS` places a six-bit *format value* into the encoded address header:

- `0x02` (`FmtByte`): normal **2:1 interleave, one-sided format**.
- `0x22` (`Fmt2Byte`): normal **2:1 interleave, two-sided format** (bit 5 set).

These are **decoded six-bit format values**; the byte physically stored in the address field is the corresponding **GCR codeword** from `NiblTbl`. Do not store or synthesize a plain `0x02`/`0x22` in place of that encoded header byte.

**[ROM]** In regular read/write/verify paths, the driver obtains its **media-side format information from the address field's format value**, not solely from the electrical fact that the installed 800K drive has two heads. It can distinguish a 400K single-sided disk in a two-sided mechanism.

**[DECISION]** Keep **drive-head capability** separate from **inserted disk's formatted side count**:

- A Sony 800K emulator remains an **800K-capable drive** in its hardware sense/status responses even when a 400K image is mounted.
- A mounted 400K image contains **80 valid side-0 tracks**; it does **not** require inventing 80 valid side-1 tracks.
- A mounted 800K image contains **160 logical track/sides**, and both heads must supply correct data.
- Track/side identity, format value and sector count in the preserved address header must be consistent with the selected track descriptor and the medium's format.

**[OPEN]** Side-selection attempts beyond the inserted medium's formatted sides are error/abnormal-read cases (Section 14), not grounds to fabricate an 800K data field on a 400K disk.

### 13.4 Physical order, interleave and logical sector identity

**[ROM]** The Plus ROM's fresh format uses fixed **2:1 physical interleave**, generated by `SONY_FILLINMARKS`. For `N` sectors, `k = ceil(N/2)`; it interleaves the lower and upper halves: `0, k, 1, k+1, ...`.

| Track class | Standard physical sequence |
|---|---|
| 12 sectors | `0 6 1 7 2 8 3 9 4 10 5 11` |
| 11 sectors | `0 6 1 7 2 8 3 9 4 10 5` |
| 10 sectors | `0 5 1 6 2 7 3 8 4 9` |
| 9 sectors | `0 5 1 6 2 7 3 8 4` |
| 8 sectors | `0 4 1 5 2 6 3 7` |

**[ROM]** The normal `Sony_ReSeek` read/write path scans valid address fields and consults a **logical sector-number bitmask**; it does not require the next recorded sector to have a particular ID. Therefore **ordering is not sector identity**.

**[DECISION]** Preserve the **observed or imported physical sequence** of valid structured sectors, even if it is not the ROM formatter's normal 2:1 permutation. This is a storage/reconstruction capability, **not** a claim that the Plus formatter itself dynamically changes its interleave. Rotate a valid physical sequence so **sector 0 is first in the logical circular buffer**, keeping all **relative successor relationships** unchanged. Every sector record stores its original encoded sector ID and is associated with a physical-order/index value; the exact byte offsets inside the 32-byte metadata remain a future ABI detail.

**[DECISION]** Build a **logical-sector-ID → record index/bit position** lookup from the preserved headers when loading a standard track. Ordinary writes update by **validated header identity**, not by assuming `record_index == sector_ID`. Verify sector ID uniqueness and the expected standard set `0..N−1` in a normal structured track; duplicate/missing IDs, inconsistent checksums and deliberately malformed layouts are not silently normalized and may require `RAW_TRACK`.

**[REFERENCE]** The sector-array's actual record order can already encode the physical sequence; the proposed order/index byte is supplementary metadata, not mandatory duplicate storage of a second entire sector table.

### 13.5 Preserved address/data headers and checksums

**[DECISION]** Each standard sector occupies exactly **735 flash bytes**:

~~~text
32-byte sector metadata (header-preserving):
    D5 AA 96 [T] [S] [H] [F] [X] DE AA  = 10-byte encoded address field
    D5 AA AD [S]                       =  4-byte encoded data header
    remaining metadata bytes           = 18 bytes (order/flags/integrity/reserved)

703-byte encoded GCR data/checksums:
    699 codewords for 12 tag bytes + 512 user bytes (6-and-2 encoded)
      4 codewords for three running data checksums (encoded)
    ------------------------------------------------------
    703 preserved codewords
~~~

**[ROM]** The address header stores **five GCR-encoded six-bit values**, in order:

| Encoded field | Underlying six-bit value |
|---|---|
| `[T]` | Low six bits of radial track number |
| `[S]` | Sector number |
| `[H]` | Side bit plus upper track bits |
| `[F]` | Format code (normal `0x02` or `0x22` before GCR encoding) |
| `[X]` | Bitwise XOR of the four **decoded six-bit** values, re-encoded through `NiblTbl` |

The data header repeats the sector ID encoded after `D5 AA AD`. The four data-checksum codewords are **already inside the 703-byte field**. The address checksum is **already inside the 10-byte address header**. Neither checksum requires a redundant generated copy.

**[DECISION]** Preserve all 14 header bytes and all 703 codewords as captured/imported. The standard-track builder may **inspect** decoded six-bit header values for consistency, but must **not routinely decode and re-encode 524 bytes of tag/user payload** in the real-time read path. An independent flash CRC/integrity field is separate from the original disk checksums. Candidate 32-byte record offsets (`10+4+1 order+1 flags+4 CRC32+12 reserved`) remain **proposed**, not a frozen on-flash ABI.

### 13.6 Deterministic standard-track reconstruction

**[DECISION]** On a clean cache miss, construct the full standard cyclic RAM GCR stream from the track descriptor and ordered 735-byte sector records; no flash reads are required during active RD playback. At least **three independent complete-track buffers of at least 12 KiB each** are available (plus separate capture rings/staging). The sector records are the backing representation; the reconstructed complete bitstream is the active RD representation.

**[ROM/DECISION]** The standard sector structure is:

~~~text
G × SYNC6                              ; exact 48-bit FF 3F CF F3 FC FF groups
STORED: D5 AA 96 [T][S][H][F][X] DE AA ; complete 10-byte address header
GENERATED: FF                          ; one-byte address OFF pad
GENERATED: SYNC6                       ; independent data synchronization
STORED: D5 AA AD [S]                   ; complete 4-byte data header
STORED: 703 GCR codewords              ; encoded tag/data + encoded data checksums
GENERATED: DE AA FF FF                 ; data-field end and trailing pads
[repeat in the preserved sector order, wrapping after final sector]
~~~

`G` is the track's **accepted** inter-sector `GapSync` count; the Plus ROM begins formatting at `G = 7` but can adjust it. The generator constructs this track **without storing gaps, SYNC6 or standard pad/slip material in SPI NOR**. It must not reinsert the 200-group format-only preamble.

**[DECISION]** Standard reconstruction pipeline:

~~~text
load track descriptor; check side, geometry, representation, version
    ↓
load 735-byte records in physical stream order
    ↓
validate expected N unique encoded sector IDs; address/data ID agreement;
         track/side/format/checksum; integrity and record bounds
    ↓
rotate ordering to sector 0; preserve original successor order
    ↓
select agreed standard circular profile, accepted GapSync, seam and bit count
    ↓
construct SYNC6, preserved headers, pads, preserved 703 bytes, data endings
    ↓
build sector-ID → bit-position table; verify circular wrap and capacity
    ↓
atomically publish complete RAM generation to RD scheduler
~~~

The standard builder owns **sector placement**, not the address/data checksum values. The read engine owns serial pulses, not sector-data synthesis. A dirty sector rewrite replaces only the relevant preserved data header and 703 codewords before rebuilding/publishing a coherent track generation (Part 4 `12).

### 13.7 Circular buffer, standard gaps, length and playback phase

**[DECISION — settled]** A reconstructed **standard** circular buffer has **sector 0 at logical position zero**, beginning at its leading address synchronization. It then contains the physical successor sequence and a ROM-compatible sync/gap seam back to sector 0. The buffer's **origin is a coordinate convention**, not an index hole, an index pulse, or a signal that the host is at the start of a revolution.

**[ROM]** For an initial `GapSync = G`, each transmitted format-loop sector consists of:

~~~text
(G − 1) × 6        bytes of additional sync
27                 bytes of ROM address/data template
703                bytes of encoded zero-data/checksums
4                  bytes of data end/off padding
----------------------------------------------
(6G + 728)         serialized bytes per sector
G = 7 → 770 bytes = 6,160 bit cells per sector
~~~

The format **write burst** begins with **200 × SYNC6 = 1,200 additional bytes**. Those bytes are a *transient lead-in*, potentially overwritten by later track recording; they are **not the stored or replayed standard loop**.

**[REFERENCE]** At `G = 7` the ordinary `N`-sector structures occupy:

| Sectors N | Sector entries only | Bit cells | Nominal format write burst including 200 sync groups |
|---:|---:|---:|---:|
| 12 | 9,240 B | 73,920 | ≈170.6 ms |
| 11 | 8,470 B | 67,760 | ≈158.0 ms |
| 10 | 7,700 B | 61,600 | ≈145.4 ms |
| 9 | 6,930 B | 55,440 | ≈132.8 ms |
| 8 | 6,160 B | 49,280 | ≈120.3 ms |

Format burst durations are **host transmission times**, not the logical revolution/track-buffer length; actual circular wrapping includes the correct residual sync/seam consistent with the standard track profile, bit rate and TACH expectations.

**[DECISION]** The selected standard profile determines the **complete circular duration, valid bit count and allowed synchronization at the wrap**. Do not invent arbitrary gap material, derive a supposedly exact bit count solely from rounded displayed TACH rates, or enlarge the track to retain the full 200-group lead-in. Use the ROM-verified structure and the profile-correct timing to reconstruct the standard loop deterministically. The *implementation* must demonstrate the expected post-format gap and return to sector 0; there is **no outstanding architectural choice about using a different track length or changing the standard geometry**.

**[DECISION]** The RD scheduler maintains a **continuously advancing phase** independently of the RAM buffer index zero. On head change, gate release or format-readback, publish the correct selected track/phase without blindly resetting the host-visible stream to sector-0's address mark. The ROM must encounter the proper **sync leading to sector 0** after the last formatted sector, allowing its immediate fit test to operate. This is stream-position bookkeeping, **not artificial mechanical rotation or rotational latency**.

**[OPEN — verification only]** Exact bench timing/tolerance of the final SYNC6 return path, phase handoff at gate release and RD pulse-width/jitter require real-IWM loopback tests; none changes the agreed standard reconstruction contract.

### 13.8 Macintosh ROM formatting and post-format verification

**[ROM]** The complete `Sony_CtlFormat` operation:

1. Determine available sides and optional one-sided `csParam = 1`.
2. Initialize `GapSync = 7`; prepare the ROM's address-mark templates.
3. For each radial track, establish the sector count `12 − floor(track / 16)`.
4. `SONY_FILLINMARKS` supplies physical interleave and the correct encoded address/data IDs, format flag and address checksum.
5. `SONY_DOFORMAT` writes one continuous gate: **200 SYNC6 groups**, then every sector entry in the selected physical order.
6. `SONY_FORMATTRACK` **immediately calls `RdAddr`** after each track/side format and expects the next address to identify **sector 0**.
7. The ROM assesses the amount of synchronization before the returned address. If too little, it can **decrease GapSync** and rewrite this same track/side (down to the ROM's `minSync = 4` limit); if ample extra synchronization is found, it may **increase GapSync for later tracks**.
8. For double-sided format, do side 1 of the same radial track before advancing to the next; finish after radial track 79.

Source: [Plus ROM formatting and gap fit, lines 2755–2804 and 2883–3078](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2755-L3078).

**[DECISION]** STM32 need not recreate the mechanical way a disk physically overwrites part of its lead-in. It receives/validates the ROM's known encoded fields and reconstructs the **ROM-acceptable logical result**. The last-to-first sync seam and RD phase **must actually allow the ROM to read sector 0 and perform its gap check**; success must not be faked by an internal software acknowledgment.

**[DECISION]** The **>=50-consecutive-SYNC6** discriminator from Part 4 `12 triggers **candidate format preparation**, not automatic success. Completed valid headers/703-byte fields, the expected number of sectors, no missing duplicates, appropriate side/format value, and the track's accepted gap/profile are needed before publishing a **coherent RAM track**. The reconstructed track is available for the ROM's immediate readback even if SPI persistence is still pending. On a ROM reformat retry, **replace** the existing candidate track rather than append sectors.

**[DECISION]** The ROM's own gap adjustment is a legitimate *format attempt/retry*, not a reason to change standard geometry. The constructed track must faithfully produce a valid sector-0 return and gap observations; no invented delay or spindle-position simulator is required.

### 13.9 Blank media, reformatting and interrupted operations

**[DECISION]** A **healthy, writable, blank** cartridge is present/format-capable even though no readable `D5 AA 96` fields exist yet. Use the standard geometry profile and provisional global timing to accept a first ROM format. Do not confuse blank media with absent media, damaged SPI contacts or failed storage hardware (Part 1 `9, Part 4 `12).

**[DECISION]** A complete valid format constructs/replaces the selected track/side's 735-byte sector records and associated profile metadata, publishes the RAM playback generation, and queues safe SPI persistence. Formatting a previously formatted track is **destructive by intention**, as on a real disk. An interrupted format need not restore the old *target track*, but **must not corrupt unrelated tracks, the directory or global calibration**; persistent failure/recovery is a separate flash transaction requirement.

**[DECISION]** For an ordinary sector overwrite on an already formatted standard track, preserve the **existing 10-byte address header and other sector records**; replace its data header/703 codewords only after validating the incoming data-field identity. Never use a format restart or sector-order assumption to rewrite unaffected sectors.

**[OPEN]** Exact dirty-buffer/power-cut ordering, wear management, cartridge removal while a format commit is pending, and unsupported malformed format attempts belong to **Section 14** and persistent-media implementation. They do not alter Section 13's standard track.

### 13.10 400K compatibility and independent full-disk Verify

**[DECISION]** The same standard 80-track/one-global-GCR-clock reconstruction engine supports both media capacities:

- **400K medium:** side 0, `0x02` decoded format value, 80 logical track records, 800 sectors (physical drive still 800K-capable).
- **800K medium:** sides 0 and 1, `0x22` decoded format value, 160 logical track records, 1,600 sectors.
- Both use the five standard sectors-per-track profiles, ROM-compatible TACH output, and ordinary 6-and-2 GCR sector fields; neither requires a second timing/calibration architecture.

**[ROM]** `Sony_CtlVerify` (control code 5) is **distinct from formatting's immediate sector-0 gap check**. It scans radial tracks from **79 down to 0**, reading each expected sector and then verifying **side 1 if the disk format indicates two sides**. It uses sector-ID maps and retries to ensure it sees every sector, instead of assuming a fixed temporal order. It reports an error when required sectors cannot be read. Source: [ROM verification, lines 3132–3245](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L3132-L3245).

**[DECISION]** The STM32 does not invent a shortcut or a special success status for Verify: it merely supplies a valid cyclic stream with correct address/data/checksums. Preserve non-2:1 valid physical order if stored; the ROM discovers IDs as it scans the track. The high-level logical medium capacity remains distinct from the electrical drive's head capability.

### 13.11 Acceptance matrix and implementation interfaces

| Test | Pass condition |
|---|---|
| Five geometry classes | Track/side sectors are exactly 12/11/10/9/8 at the expected radial ranges |
| 400K single-sided ROM format | All 80 side-0 tracks formatted; `0x02` decoded header format; no fabricated side-1 disk data |
| 800K double-sided ROM format | Both heads formatted, 160 track/sides; `0x22` decoded format |
| Normal 2:1 interleave | Each profile emits exact ROM sector order and addresses/checksums |
| Alternative valid sector permutation | Preserved as captured/imported; logical buffer rotates to sector 0 without changing successor order |
| Encoded headers | Full 10-byte address and 4-byte data headers retained unchanged; sector ID/side/track/format/checksum checked |
| Encoded user data/checksums | All **703** codewords retained unchanged; 699+4 boundaries verified |
| Standard reconstruction | Canonical SYNC6, correct `GapSync`, one address `FF` pad, `DE AA FF FF` data ending |
| Circular origin and phase | Buffer starts at sector 0; RD clock/phase is independent; correct seam and wrap; no index pulse |
| Post-format ROM check | After gated format, `RdAddr` finds sector 0 with acceptable measured sync, including legitimate retry |
| Independent `CtlVerify` | 400K and 800K full-disk verification finds every required sector |
| Read/write coexistence | Ordinary writes preserve address and all other records; reading uses coherent RAM generation |
| Three-track cache | >=3 complete >=12-KiB buffers plus capture staging, with no active-stream corruption during flash work |
| Standard TACH and GCR clock | Each profile's TACH matched and global nominal 489,600-bit/s GCR clock bench-measured |
| Blank media / repeat format | First format works without pre-existing sectors; retries do not append duplicate sector maps |
| RAW_TRACK isolation | No silent normalization of custom gaps/headers/protection or reinterpretation as standard tracks |

**Interface ownership:**

- [Part 2 `10](06-part-2-serial-gcr-read.md): electrical RD pulses, bit ordering and byte synchronization.
- [Part 3 `11](06-part-3-media-architecture.md): representation IDs, 735-byte records, descriptors, >=3 cache buffers, global calibration and allocation.
- [Part 4 `12](06-part-4-serial-gcr-write.md): WR gate, both-edge capture, 50-SYNC6 classification, standard data-field replacement and SPI flush scheduling.
- **This `13:** standard geometry, side/format semantics, physical ordering, fixed-format sector reconstruction, circular loop and ROM format/verify behaviour.
- **Future `14:** errors, unusual sequences, interrupted operations and safe recovery.

### 13.12 Settled decisions and remaining engineering gates

**Settled [DECISION]:**

- Reproduce the complete standard Macintosh 400K/800K **host-visible GCR track**, not a new serialization format.
- 80 radial positions; 12/11/10/9/8 sectors per side in five 16-track classes; standard TACH lookup and **one global** GCR timing calibration.
- 400K uses **side 0** and normal decoded format `0x02`; 800K uses two sides and normal `0x22`. Electrical 800K head capability is not identical to the mounted disk's formatted sides.
- ROM-generated tracks use **2:1 physical interleave**. Preserve the **actual valid recorded sector order**; identify sectors from their stored headers, normalize circular RAM origin to **sector 0** without changing successor order.
- Store **32 metadata + 703 unchanged encoded GCR codewords** per sector, including preserved complete address/data headers. Rebuild SYNC6, gaps and fixed pad/slip regions in a complete RAM track; do not persist the transient 200-group preamble.
- Standard **circular duration, gaps and wrap** reproduce the ROM-compatible track profile. Buffer index zero and live RD phase are different coordinates. No simulated mechanics or arbitrary delay.
- Format must support immediate genuine ROM **sector-0** gap/address readback, possible gap retries and a separate full-disk Verify; RAW_TRACK remains unnormalized for unknown/nonstandard media.
- At least **three >=12-KiB complete-track buffers** plus separate edge-capture staging and asynchronous flash persistence, subject to safe ejection and durability.

**Engineering validation/implementation tasks (not reopened architectural decisions):**

- Record the ROM-compatible circular wrap/gap and first post-format readback waveforms on the target Macintosh; prove timer jitter and phase continuity against the real IWM.
- Establish exact 32-byte sector metadata and track-descriptor binary ABI/CRC, with standard physical-order lookup and accepted `GapSync` fields.
- Verify alternative sector permutations and 400K single-sided operation with real ROM test media.
- Implement protected flash transactions, dirty-generation/buffer exhaustion policy, power-fail/eject recovery and RAW_TRACK write limitations in later sections.

**Document history**

- **2026-11-09:** Created Part 5 / Section 13 from the settled Sections 10–12 architecture and inspected the Plus ROM's `Sony_CtlFormat`, `SONY_FILLINMARKS`, `SONY_DOFORMAT`, `SONY_FORMATTRACK`, ordinary `Sony_ReSeek` and separate `Sony_CtlVerify`. Fixed normal geometry, side/format semantics, physical order preservation, logical sector-0 origin and reconstructed ROM-compatible cyclic track while retaining hardware acceptance tests.
