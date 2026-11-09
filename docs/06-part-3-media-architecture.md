# Sony MFD-51W Drive Interface Protocol — Part 3: Media Architecture and Settled Design Decisions

**Document:** 06 — Part 3  
**Protocol section:** **11 — Shared structured-GCR/raw-track media, timing and cartridge architecture**  
**Status:** Consolidated architectural decisions; implementation details marked **[PROPOSED]** or **[OPEN]**  
**Target:** Macintosh Plus M0001AP, original ROM `.Sony` driver and IWM, host-facing Sony MFD-51W 800K mechanism  
**Date:** 9 November 2026

> **Where this part fits:** [Part 1, Sections 1–9](06-part-1-drive-interface.md) documents the host-facing interface, commands, status and TACH; [Part 2, Section 10](06-part-2-serial-gcr-read.md) documents serial reading and the exact freshly formatted GCR track stream. **This Part 3, Section 11** is the common architecture used by **both** reading and writing. **[Part 4, Section 12](06-part-4-serial-gcr-write.md)** develops the detailed serial writing procedure; later topics begin at Section 13. This is **not** a replacement for either the read or write electrical specifications.
>
> **Evidence labels:** **[ROM]** observed Plus ROM assembly; **[SONY]** original drive specification; **[DECISION]** agreed project requirement; **[PROPOSED]** candidate implementation, not yet approved as a flash format or hardware design; **[OPEN]** requires engineering, hardware evidence, or a further decision.

## Contents

- [11.1 Responsibility boundary and data flow](#111-responsibility-boundary-and-data-flow)
- [11.2 Structured GCR, raw-track compatibility and synchronization](#112-structured-gcr-raw-track-compatibility-and-synchronization)
- [11.3 Global bit-cell timing and SYNC6 calibration](#113-global-bit-cell-timing-and-sync6-calibration)
- [11.4 Logical track descriptors, without a motor-zone model](#114-logical-track-descriptors-without-a-motor-zone-model)
- [11.5 TACH is independently required by the Plus ROM](#115-tach-is-independently-required-by-the-plus-rom)
- [11.6 Track playback, sector writes and RAM buffers](#116-track-playback-sector-writes-and-ram-buffers)
- [11.7 SPI-flash layout, 735-byte records and centralized metadata](#117-spi-flash-layout-735-byte-records-and-centralized-metadata)
- [11.8 Versioning, forward extension and lossless metadata upgrades](#118-versioning-forward-extension-and-lossless-metadata-upgrades)
- [11.9 Cartridge formatting, external programming and reuse](#119-cartridge-formatting-external-programming-and-reuse)
- [11.10 Constraints and interfaces for subsequent parts](#1110-constraints-and-interfaces-for-subsequent-parts)
- [11.11 Decisions, proposals and open questions](#1111-decisions-proposals-and-open-questions)

---

## 11. Shared structured-GCR/raw-track media, timing and cartridge architecture


### 11.1 Responsibility boundary and data flow

**[DECISION]** Emulate the Sony drive's **host-visible electrical and logical behaviour**, using the original Macintosh ROM, MC68000, IWM and VIA. Do not emulate physical mechanics, acoustic delays, spin-up, artificial seek time or rotation for realism. Electrical waveforms, write gating, RD status/TACH and timing requirements remain mandatory.

**[ROM]** The Macintosh ROM performs 6-and-2 encoding/decoding. The IWM serializes encoded bytes into WR level transitions and deserializes RD pulses. The STM32 observes the electrical stream, **not** a byte-wide GCR interface. For ordinary writes the ROM writes one data field; for formatting it writes the complete track/side.

**[DECISION — revised 2026-11-09]** Standard 400K/800K media use **structured encoded-GCR sectors** in SPI NOR, not a permanently stored full physical bitstream. Each standard sector has **32 metadata bytes plus 703 stored GCR codewords** (699 encoded tag/data + four encoded data checksums). Its complete address and data headers are retained in the 32-byte metadata. The STM32 **reconstructs SYNC6, gap, fixed pad/slip patterns, interleave and the circular track in RAM**. The 703 payload/checksum codewords are preserved; routine live 6-and-2 payload decoding/re-encoding is unnecessary. An alternate **RAW_TRACK** representation preserves unknown/nonstandard track bits without normalization.

~~~text
NORMAL WRITE (standard track):
ROM 6-and-2 encoder → IWM WR level transitions → qualified STM32 both-edge capture
    → recognize/validate sector field → update 735-byte sector record in RAM
    → refresh affected prepared cyclic track → asynchronous flash persistence

NORMAL READ (standard track):
SPI NOR structured GCR records → reconstruct full cyclic bitstream in RAM
    → timed RD pulses → IWM byte assembly → ROM 6-and-2 decoder

NONSTANDARD READ:
SPI NOR RAW_TRACK packed bits → RAM unchanged → timed RD pulses
~~~

**[DECISION]** Track buffering and deterministic WR capture/RD output are independent of flash erase/program scheduling. For standard tracks, preserving canonical decoded *structure* is different from changing encoded payload bytes. Nonstandard/raw-only tracks must never be silently canonicalized.

Sources: [Plus ROM sector write, lines 2547–2719](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2547-L2719); [format path, lines 2883–3078](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2883-L3078).


### 11.2 Structured GCR, raw-track compatibility and synchronization

**[DECISION]** Two representation modes share the same versioned track directory:

- **STRUCTURED_GCR (default):** a sector array of **735-byte records**, comprising 32 bytes of metadata (including preserved encoded headers) and 703 bytes of encoded data/checksums. No physical gaps, SYNC6, standard address pad or data-ending padding are persistently stored as part of the sector array.
- **RAW_TRACK:** bit-packed complete track data, a valid bit count and a wrap point. Replay precisely as recorded, without synthetic gaps/headers or automatic sync normalization. Nonstandard write support remains a separate engineering gate.

**[DECISION]** Standard-track reconstruction produces this ROM-compatible sequence, including the established 2:1 interleave:

~~~text
G x SYNC6 → stored address: D5 AA 96 [T] [S] [H] [F] [X] DE AA
          → generated FF address pad
          → generated SYNC6
          → stored data header: D5 AA AD [S]
          → stored 703 encoded GCR tag/data/checksum bytes
          → generated DE AA FF FF
          → next address synchronization / circular seam
~~~

The retained 10-byte address field contains its encoded checksum; the four **data** checksum codewords are the last four of the 703-byte payload. The 4-byte data header contains the separately encoded sector ID. Preserve these values exactly, verifying consistency when constructing/accepting standard tracks.

The canonical self-synchronization group remains exactly:

~~~text
SYNC6 = FF 3F CF F3 FC FF (48 transmitted bits, MSB first)
      = 1111111100 1111111100 1111111100 1111111100 11111111
~~~

**[ROM]** Formatting sends **200 SYNC6 groups as a transient write lead-in**. This is **not** a permanent 200-group prefix in the resulting cyclic track. Per-sector `GapSync` begins at seven groups and may be adjusted by ROM readback. The accepted gap policy and the reconstructed track's seam/wrap length must be recorded at track level; do not assume seven forever.

**[DECISION]** Generate all standard SYNC6/gap/fixed pad and slip bytes only in known standard regions. Rebuild the complete RAM track **before** exposing it on RD; do not insert/delete bits during active serial playback. For RAW_TRACK the exact stored bit sequence is authoritative. [Part 2 `10.6](06-part-2-serial-gcr-read.md#106-full-sector-structure--exact-macintosh-plus-freshly-formatted-reference) is the ROM stream reference.

**[OPEN]** Exact circular seam/gap fitting, accepted bit length, special/partially formatted tracks and RAW_TRACK write semantics require hardware/ROM tests; the standardized representation is not bit-perfect archival storage for unknown copy protection.

### 11.3 Global bit-cell timing and SYNC6 calibration

**[DECISION]** For the project, the nominal GCR serial rate is **489,600 bit cells/second**, corresponding to **2.042484 µs per cell**, pending confirmation using a logic analyser. This nominal rate is the **same for all tracks**; the project does not use five independent GCR clocks.

**[DECISION]** A **single global timing calibration record** applies to the complete disk/cartridge. **No per-track or per-side bit-rate calibration** is stored. The measured value is not a sector-layout field.

**[DECISION]** The Macintosh's normal **format operation** is the preferred *automatic operational calibration source*. The Plus ROM transmits a known `SYNC6` lead-in of **200 × 48 bit cells** at the start of a track format (9,600 cells). The STM32 can recognize repeated runs and timestamp WR transitions against its local timer, infer the bit-cell duration and reduce measurement noise by averaging many valid intervals or entire groups. Known `SYNC6` transitions include runs of 1 and 3 bit-cell intervals.

**[PROPOSED]** The STM32 maintains a provisional nominal clock for startup/first-format compatibility, then accepts a timing calibration only when validated (pattern, sample count, measured jitter/variance, bounds and integrity). Record a usable period or integer timer ratio rather than relying on unqualified floating-point serialization. The exact algorithm and acceptance criteria remain open.

**Critical limits:** SYNC6 timing calibration measures the **incoming IWM write stream relative to the STM32 timer**. It cannot by itself prove RD voltage levels, pulse width/polarity, tri-state behavior, write-gate setup/hold, oscillator stability or every host electrical tolerance. These require **design validation**, but no user-facing manual oscilloscope calibration should be necessary once hardware is validated. The initial format must function on the nominal timing baseline to obtain the measurement.

**[DECISION]** Store timing calibration in **global cartridge metadata**, distinct from the raw disk image and track directory. The external programmer must preserve it when changing a disk image. **[OPEN]** Calibration portability between different emulator units and host machines, oscillator drift and a fallback/un-calibrated policy require explicit rules.


### 11.4 Logical track descriptors, without a motor-zone model

**[DECISION]** One logical track/side is the storage, caching and playback unit. Its directory descriptor identifies the representation and its allocation:

~~~text
LogicalTrack:
    track_number, side                 (0–79; side 0/1 as applicable)
    representation                     (STRUCTURED_GCR or RAW_TRACK)
    sector_count                       (standard: 12/11/10/9/8)
    data_offset, allocation_bytes, used_bytes
    valid_bit_length, wrap_phase       (reconstructed or raw)
    gap_sync / seam_policy            (for standard reconstruction)
    interleave / physical_sector_order
    format_flags, valid/formatted/dirty-generation metadata
    integrity + version information
    optional verified field/bit position map
~~~

For **STRUCTURED_GCR**, `data_offset` points to the contiguous sector-record array; `used_bytes = sector_count × 735` for this revision. The RAM playback track is larger than the flash sector array and has a separately calculated valid bit count. For **RAW_TRACK**, `data_offset` points to packed bits and `valid_bit_length` limits the final byte/wrap point. Never confuse a physical flash allocation length with reconstructed circular bit length.

**[DECISION]** No motor zone or circumference simulation. The MCU consumes the track descriptor's logical sector count and bit length, using **one global calibrated GCR bit-cell rate**. The descriptor's sector count also selects the standard TACH status frequency (`11.5), independently of GCR timing. Never invent an index pulse or restart read phase on a mere status selection.

**[OPEN]** Exact binary descriptor ABI, allowed `GapSync` bounds, seam computation and nonstandard track size limits remain to be defined and verified.

### 11.5 TACH is independently required by the Plus ROM

**[ROM — VERIFIED]** The 800K/new-interface Sony drive omits legacy *motor-speed adjustment*, but the Macintosh Plus **still measures TACH** in the normal `Sony_ReSeek` path. It calls `SONY_GETFREQ1` before the `NewIntf` test that skips PWM adjustment; absence of TACH pulses can cause a timeout, and out-of-range speed may cause a speed-adjustment error. Formatting's separate `SONY_SPDSEEK` path **does** skip speed measurement for 800K. Full evidence and host timings: [Part 1, §§8.3–8.6](06-part-1-drive-interface.md#83-five-speed-zones-physical-800k-mechanism-versus-electronic-model); [ROM `Sony_ReSeek`, lines 832–882](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L832-L882).

**[DECISION]** The STM32 must generate a genuine, time-varying **TACH status waveform** when selected by the host. The clock phase should not restart merely because the Macintosh selects the TACH source on RD. TACH requires neither a simulated rotating mechanism nor a separate GCR bit-cell calibration.

For conventional formatted tracks, derive the **target TACH frequency from the descriptor's sector count**, using the verified 800K host expectations:

| Sector count on selected track | Nominal TACH rate (pulses/s) | Legacy physical explanation only |
|---:|---:|---|
| 12 | 394 Hz | tracks 0–15 |
| 11 | 429 Hz | tracks 16–31 |
| 10 | 472 Hz | tracks 32–47 |
| 9 | 525 Hz | tracks 48–63 |
| 8 | 590 Hz | tracks 64–79 |

The original drive emits **60 TACH pulses per physical revolution**, but the emulator implements a **timer-generated status signal**, not a spinning medium. This table is an **electrical compatibility lookup**, not a physical-zone simulation. **No TACH-specific calibration record is stored in each track.**

**[OPEN]** The cyclic read-track bit length and the selected TACH frequency must remain acceptably coherent; a field's length alone is **not** a sufficient guarantee of the ROM's required TACH frequency. The final standard-track generator must verify both constraints when constructing or importing tracks. The exact treatment of nonstandard/custom sector counts and stopped-TACH levels is deferred.


### 11.6 Track playback, sector writes and RAM buffers

**[ROM]** The ordinary Plus ROM write modifies **one data field**, after reading the target address; `Sony_WrData` does not rewrite its address field. `SONY_DOFORMAT` writes an entire track/side and the ROM immediately checks sector 0, potentially retrying with a different `GapSync`.

**[DECISION]** Provision **at least three independently managed complete-track RAM buffers of at least 12 KiB each** (minimum 36 KiB total), plus separate DMA/timer capture rings, page staging and bookkeeping. The buffers may serve ACTIVE/READABLE, INCOMING/RECONSTRUCTING and DIRTY/FLUSHING roles, but roles are dynamic, not permanent. A buffer must not be reused while it contains the only authoritative copy of a dirty or actively streamed track.

**[DECISION]** Standard reads operate from a **fully reconstructed cyclic bitstream in RAM**; RAW_TRACK reads use the original packed bits unchanged. The read scheduler and WR capture must remain deterministic while SPI flash is busy. A current track's phase and verified sector map identify the candidate target on ordinary writes; incoming sector ID/field structure must validate that association.

**[DECISION]** For standard sector writes, use **structural data-field replacement** of preserved encoded data/checksum/header values, with generated canonical sync and gaps. No exact physical flux splice reconstruction is required for the standardized mode; do not modify other sector records or the existing address field. For RAW_TRACK, exact bit-splice policy remains [OPEN]. For a complete format, construct a fresh track/side record array and an appropriate cyclic RAM track, not a permanent 200-SYNC6 prefix.

**[DECISION]** SPI flash persistence is an **asynchronous worker** decoupled from capture, with erase/program work permitted during subsequent host operations. Dirty cache is not durable. Track changes alone may not trigger the only commit, because the Macintosh can finish a write and eject or power down without changing track. Implement an explicit safe-eject/durability/error policy; with only three buffers, backlog/buffer exhaustion must be detected and cannot silently discard accepted writes.

**[OPEN]** Final hardware edge alignment, atomic publication, flash commit/recovery, performance under continuous sector writes, underrun and buffer saturation require validation in [Part 4 `12](06-part-4-serial-gcr-write.md).


### 11.7 SPI-flash layout, 735-byte records and centralized metadata

**[DECISION]** The removable cartridge target is **16 Mbit = 2 MiB = 2,097,152 bytes** SPI NOR. The standard-sector persistence contract is **32 bytes metadata + 703 GCR bytes = 735 bytes per sector** (699 tag/user-data codewords and four checksum codewords).

**[DECISION — record size and retained fields]** The 32-byte sector metadata retains the complete **10-byte address field** (`D5 AA 96 [T][S][H][F][X] DE AA`, checksum included) and complete **4-byte data header** (`D5 AA AD [S]`). The remaining 18 bytes support physical ordering, flags, integrity and future extension. **[PROPOSED — nonfrozen ABI]** A useful candidate is: address 10 + data header 4 + physical order 1 + flags 1 + CRC32 4 + reserved 12 = 32. The CRC should cover at least both stored headers and the 703 encoded bytes. Record size and 703-byte payload are settled; exact offsets/CRC scheme are not yet approved as a firmware ABI.

**[DECISION]** For standard tracks use **individually erase-aligned 4-KiB allocation extents** sized by actual sector count. Two 4-KiB sectors hold **8–11 standard sectors**, but **12 standard sectors need three** (8,820 > 8,192 bytes). Each track descriptor records its offset, used length and full allocation. Separate metadata directories must not be mixed into those erase units.

| Sectors | Used bytes (735 × sectors) | 256-byte pages (ceil) | 4-KiB erase units | Allocation |
|---:|---:|---:|---:|---:|
| 12 | 8,820 | 35 | 3 | 12 KiB |
| 11 | 8,085 | 32 | 2 | 8 KiB |
| 10 | 7,350 | 29 | 2 | 8 KiB |
| 9 | 6,615 | 26 | 2 | 8 KiB |
| 8 | 5,880 | 23 | 2 | 8 KiB |

For a standard two-sided 800K disk there are 32 track/sides in each sector-count class (160 total): **1,600 sectors × 735 = 1,176,000 bytes of records**. Dedicated erase-aligned extents require **1,441,792 bytes = 1,408 KiB**, leaving **655,360 bytes = 640 KiB** of the 2-MiB medium before global metadata, directory copies, raw-track alternatives, spare blocks and journals.

**[DECISION]** Maintain separate **global media/calibration metadata**, a centralized **extensible logical-track directory**, and **track data extents**, referenced by offset/length and representation mode. Reserve generous directory extension space and support schema upgrades without relocating/wiping referenced track data solely to add metadata fields.

~~~text
2 MiB SPI NOR (conceptual; offsets unassigned)
    Superblock, global identity and one GCR timing calibration
    Versioned centralized track directory (+ optional recoverable generation)
    Independent erase-aligned structured-sector / raw-track allocations
    Spare extents, journal and recovery capacity
~~~

**[PROPOSED]** A 512-byte directory descriptor per track/side is an illustrative expansion allowance: 160 × 512 = 80 KiB; two generations = 160 KiB. This consumes part of the 640-KiB balance and does **not** imply that the final safe-write/journal scheme fits without analysis.

**[REFERENCE TIMING — not a final flash selection]** The Microchip [SST26VF016B datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/SST26VF016B-2.5V-3.0V-16-Mbit-Serial-Quad-IO-SQI-Flash-Memory-Data-Sheet-DS20005262F.pdf) lists maximum 25 ms per 4-KiB erase and 1.5 ms per 256-byte page program. Sequential erase + page-program upper-bound **internal operation** budgets are therefore **127.5 / 98.0 / 93.5 / 89.0 / 84.5 ms** for 12/11/10/9/8-sector tracks, respectively. Excludes SPI transfers, command/polling, verification, metadata commits, contention and retries. The actual selected chip must be benchmarked.


### 11.8 Versioning, forward extension and lossless metadata upgrades

**[DECISION]** A previously programmed cartridge must allow **metadata/schema upgrades without loss of stored sector records or raw-track content**. Version superblocks, directories and record representations independently; never reinterpret unknown format versions as if they were current.

**[PROPOSED]** Use CRC/integrity and recoverable directory generations. A safe metadata-only migration validates old extents, builds a new directory without modifying referenced data, verifies it, atomically activates it using a flash-safe procedure, and only then reclaims old metadata. The specific atomic-update algorithm is **[OPEN]**.

**[DECISION]** Separate representation IDs for **STRUCTURED_GCR (32+703)** and **RAW_TRACK (bit-packed length)**, and a separately versioned global calibration record. Upgrading metadata should preserve both existing GCR contents and the global calibration. A conversion between representations is a distinct operation with explicit fidelity limits.

**[OPEN]** Power-cut atomicity, protection of unaffected erase units, wear accounting and safe recovery require the actual flash choice and an on-media transaction design.


### 11.9 Cartridge formatting, external programming and reuse

**[DECISION]** The reference workflow:

~~~text
1. Insert healthy blank SPI-flash cartridge in the emulator.
2. Macintosh ROM/IWM formats it; STM32 recognizes the 200-SYNC6 preamble,
   captures preserved GCR headers and 703-byte codeword fields per sector,
   reconstructs/validates standard tracks, and calibrates one global GCR clock.
3. Safely commit and eject the cartridge.
4. External programmer imports an image and creates STRUCTURED_GCR records
   for standard tracks, or RAW_TRACK data where exact bitstream is available
   and necessary. Preserve cartridge global calibration and track metadata.
5. Reinsert; STM32 reconstructs standard tracks into RAM (or replays raw)
   and serves timed RD pulses with appropriate status/TACH.
~~~

**[DECISION]** One successful Macintosh format is a preferred calibration/initialization route, **not** a prerequisite for re-importing every image. After calibration, an external programmer can replace images without discarding that calibration. It must not falsely claim lossless nonstandard reconstruction from decoded 512-byte-only images.

**[OPEN]** First-use provisional timing, host-to-host calibration portability, decoded/tagged/raw input conversion, and safe ejection/commit mechanics need their own implementations.


### 11.10 Constraints and interfaces for subsequent parts

**Part 1 / ``1–9:** Defines electrical pins, status/control, seeks, READY/TACH, eject and insertion. TACH depends on logical sector count, not on simulated motor geometry.

**Part 2 / `10:** Defines actual RD pulses and exact ROM-standard serial GCR track reference. For STRUCTURED_GCR, the read engine consumes the **fully reconstructed RAM cyclic track**; for RAW_TRACK it consumes the unchanged packed-bit RAM track.

**[Part 4 / `12](06-part-4-serial-gcr-write.md):** Defines both-edge WR capture, the 50-consecutive-SYNC6 **candidate format** discriminator, encoded field validation/structured replacement, >=3 complete-track buffers, asynchronous persistence and unresolved failure handling. It must preserve immediate post-format sector-0 readback and not claim write success merely from gate release.

**Later work ``13–18:** Format/400K edge cases; faults; electrical timing; STM32 peripherals; acceptance; decision log. Separate media/programmer design may expand the on-flash ABI, but must honour these agreed architectural requirements.


### 11.11 Decisions, proposals and open questions

**Settled [DECISION]:**

- Unchanged original ROM/IWM and host-visible Sony electrical semantics; no artificial mechanical latency.
- **Structured encoded-GCR standard tracks** use 32-byte sector metadata (including stored 10-byte address and 4-byte data headers) plus **703 unchanged GCR data/checksum bytes**; alternate untouched RAW_TRACK storage is supported as a representation.
- Generate standard SYNC6, gaps, fixed pad/slip patterns and cyclic track in RAM. Store accepted gap policy/track valid bit length and do not retain the transient 200-group preamble as part of the sector array.
- At least **three complete >=12-KiB RAM track buffers**, additional capture DMA rings and a flash worker independent of real-time serial capture.
- Standard writes structurally update only the identified sector data field; RAW_TRACK writes remain a separate fidelity question.
- Dedicated erase-aligned track extents and offset/used/allocation directory descriptors, with versioned, extensible centralized metadata and one global GCR calibration.
- 2-MiB cartridge target; standard full-disk allocation budget 1,408 KiB plus metadata/spares; standard 12-sector tracks require three 4-KiB erase sectors.
- TACH frequency follows descriptor sector count; one global GCR bit-cell timing nominally 489,600 bit/s, bench confirmation pending.
- Macintosh format/calibration, subsequent external image conversion and reuse remain supported. Eject/dirty writes must be managed safely.

**Agreed at architectural level; exact binary/implementation details open:**

- 32-byte sector metadata contains headers, ordering/flags/integrity/reserve; the illustrated byte layout and integrity algorithm are a proposed ABI.
- ROM-derived candidate format recognition after >=50 consecutive SYNC6 groups within one qualified gate; final acceptance requires validating the complete formatted stream.
- Three-buffer cache states/scheduling, async flash erase/page-program and dirty-generation ownership; final backpressure/recovery rules are open.

**Still open:**

- Specific STM32 and flash device, executable capture/DMA and RD hardware timing, exact sector/header validation, full-track seam and `GapSync` fitting.
- Flash atomicity, interrupted-sector-write vs interrupted-format semantics, wear, safe eject, power-off recovery and no-silent-loss buffer exhaustion.
- Final on-flash addresses, CRC and metadata ABI, coexistence of optional raw tracks and spare/journal extents.
- RAW_TRACK write splices and imported copy-protected/nonstandard-image fidelity; image source tag availability and calibration portability.

**Document history**

- **2026-11-09 (write architecture review):** Approved structured 32+703-byte standard-sector records with both encoded headers preserved, reconstructed standard gaps/SYNC6/track, optional RAW_TRACK representation, >=3 complete 12-KiB buffers, erase-aligned capacity and reference timing budgets; revised Parts 2/4 accordingly. Binary ABI and crash-safety remain open.

- **2026-11-09:** Linked developed [Part 4 / Section 12](06-part-4-serial-gcr-write.md) for the detailed write path; retained Section 11 as the shared architecture and kept unresolved hardware/flash details open.
- **2026-11-09:** Created **06 — Part 3 / §11** to consolidate agreed media, SYNC6 calibration, logical track, TACH and external programmer architecture. Renumbered planned **serial writing to Part 4 / §12**, and deferred later protocol sections to **§§13–18**. No binary flash layout or write-timing algorithm is implied by this documentation reorganization.
