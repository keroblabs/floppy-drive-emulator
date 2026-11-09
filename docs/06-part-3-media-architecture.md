# Sony MFD-51W Drive Interface Protocol — Part 3: Media Architecture and Settled Design Decisions

**Document:** 06 — Part 3  
**Protocol section:** **11 — Shared raw-GCR media, timing and cartridge architecture**  
**Status:** Consolidated architectural decisions; implementation details marked **[PROPOSED]** or **[OPEN]**  
**Target:** Macintosh Plus M0001AP, original ROM `.Sony` driver and IWM, host-facing Sony MFD-51W 800K mechanism  
**Date:** 9 November 2026

> **Where this part fits:** [Part 1, Sections 1–9](06-part-1-drive-interface.md) documents the host-facing interface, commands, status and TACH; [Part 2, Section 10](06-part-2-serial-gcr-read.md) documents serial reading and the exact freshly formatted GCR track stream. **This Part 3, Section 11** is the common architecture used by **both** reading and writing. **Part 4, Section 12** is reserved for the detailed serial writing procedure; later topics begin at Section 13. This is **not** a replacement for either the read or write electrical specifications.
>
> **Evidence labels:** **[ROM]** observed Plus ROM assembly; **[SONY]** original drive specification; **[DECISION]** agreed project requirement; **[PROPOSED]** candidate implementation, not yet approved as a flash format or hardware design; **[OPEN]** requires engineering, hardware evidence, or a further decision.

## Contents

- [11.1 Responsibility boundary and data flow](#111-responsibility-boundary-and-data-flow)
- [11.2 Raw-GCR tracks and canonical synchronization](#112-raw-gcr-tracks-and-canonical-synchronization)
- [11.3 Global bit-cell timing and SYNC6 calibration](#113-global-bit-cell-timing-and-sync6-calibration)
- [11.4 Logical track descriptors, without a motor-zone model](#114-logical-track-descriptors-without-a-motor-zone-model)
- [11.5 TACH is independently required by the Plus ROM](#115-tach-is-independently-required-by-the-plus-rom)
- [11.6 Track playback, sector writes and RAM buffers](#116-track-playback-sector-writes-and-ram-buffers)
- [11.7 SPI-flash layout and centralized metadata](#117-spi-flash-layout-and-centralized-metadata)
- [11.8 Versioning, forward extension and lossless metadata upgrades](#118-versioning-forward-extension-and-lossless-metadata-upgrades)
- [11.9 Cartridge formatting, external programming and reuse](#119-cartridge-formatting-external-programming-and-reuse)
- [11.10 Constraints and interfaces for subsequent parts](#1110-constraints-and-interfaces-for-subsequent-parts)
- [11.11 Decisions, proposals and open questions](#1111-decisions-proposals-and-open-questions)

---

## 11. Shared raw-GCR media, timing and cartridge architecture

### 11.1 Responsibility boundary and data flow

**[DECISION]** Emulate the Sony drive's **host-visible electrical and logical behaviour**. The original Macintosh ROM, MC68000, IWM and VIA remain in use. Do not introduce mechanically realistic delays, a motor simulator or a virtual floppy controller; retain electrical waveforms, status semantics, write gating, and timing required by the host.

The **Macintosh ROM implements 6-and-2 data encoding and decoding**. For ordinary writes, `Sony_WrData` progressively encodes the sector payload and presents encoded bytes to the IWM; the **IWM serializes** these bytes as write-data transitions. The emulator receives the transitions, not a byte-wide GCR bus. During reads, the STM32 supplies timed read pulses to the IWM; the IWM assembles encoded bytes and the ROM decodes them.

~~~text
NORMAL WRITE:
Macintosh sector buffer + ROM 6-and-2 encoder
    → IWM write serialization
    → Sony connector WR / write gate
    → STM32 timestamped transition capture
    → bit-packed GCR track update in RAM
    → persistence to removable SPI NOR flash

NORMAL READ:
Removable SPI NOR flash: bit-packed raw-GCR track
    → STM32 RAM track buffer and timed pulse generator
    → Sony connector RD
    → IWM serial-to-byte assembly
    → ROM 6-and-2 decoder
    → Macintosh sector buffer
~~~

**[DECISION]** The STM32 does **not** routinely encode or decode the 524 decoded sector bytes for the live read/write path. It may inspect recognizable GCR prologues, sync regions and encoded address values to build/verify a track map. **External image conversion**, discussed in `11.9`, can perform full 6-and-2 encoding outside the timing-critical firmware.

Sources: [ROM `Sony_ReSeek` sector-select/write path, lines 884–1012](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L884-L1012); [ROM `Sony_WrData`, lines 2547–2719](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2547-L2719).

### 11.2 Raw-GCR tracks and canonical synchronization

**[DECISION]** The removable medium stores **raw 6-and-2 GCR in a packed-bit representation**, preserving the order and alignment of every serialized bit cell. Physical SPI-flash bytes are **containers**, not necessarily byte-aligned GCR codewords. The selected track is treated as a **cyclic bitstream**, whose exact valid bit length is recorded separately.

**[DECISION]** The STM32 must have a built-in canonical self-synchronization generator for standard formatted tracks:

~~~text
SYNC6 as six transmitted ROM bytes: FF 3F CF F3 FC FF

Serial bits, most-significant first:
11111111 00111111 11001111 11110011 11111100 11111111
= (1111111100) × 4 + 11111111
~~~

The ROM writes `SYNC6` through the IWM; the IWM does **not** synthesize this pattern on the Sony connector. An address field begins `D5 AA 96` and a data field begins `D5 AA AD`, each with its own preceding synchronization. The freshly formatted reference, exact `DE AA FF` address pad, `DE AA FF FF` data end, `GapSync` repetition and physical interleave are documented in [Part 2, §10.6](06-part-2-serial-gcr-read.md#106-full-sector-structure--exact-macintosh-plus-freshly-formatted-reference).

**[DECISION]** Canonical sync may be **constructed or repaired only in known synchronization regions of an explicitly standard-format track**, without modifying the preserved encoded sector payload or arbitrarily changing its bit length. A region map/validation step must establish the intended boundaries. An unknown, nonstandard or copy-protected raw track is replayed as recorded; it must not be silently normalized.

**[ROM]** The formatter sends 200 `SYNC6` groups as an initial lead-in, but this is **not** a permanent 200-group prefix to every finished cyclic track: it can be overwritten as the formatting operation passes one revolution. Per-sector gaps are controlled by the ROM's `GapSync` value, initially seven groups before the address mark and subject to fitting adjustments. Preserve the distinction between the **write sequence** and the **final cyclic medium contents**.

### 11.3 Global bit-cell timing and SYNC6 calibration

**[DECISION]** For the project, the nominal GCR serial rate is **489,600 bit cells/second**, corresponding to **2.042484 µs per cell**, pending confirmation using a logic analyser. This nominal rate is the **same for all tracks**; the project does not use five independent GCR clocks.

**[DECISION]** A **single global timing calibration record** applies to the complete disk/cartridge. **No per-track or per-side bit-rate calibration** is stored. The measured value is not a sector-layout field.

**[DECISION]** The Macintosh's normal **format operation** is the preferred *automatic operational calibration source*. The Plus ROM transmits a known `SYNC6` lead-in of **200 × 48 bit cells** at the start of a track format (9,600 cells). The STM32 can recognize repeated runs and timestamp WR transitions against its local timer, infer the bit-cell duration and reduce measurement noise by averaging many valid intervals or entire groups. Known `SYNC6` transitions include runs of 1 and 3 bit-cell intervals.

**[PROPOSED]** The STM32 maintains a provisional nominal clock for startup/first-format compatibility, then accepts a timing calibration only when validated (pattern, sample count, measured jitter/variance, bounds and integrity). Record a usable period or integer timer ratio rather than relying on unqualified floating-point serialization. The exact algorithm and acceptance criteria remain open.

**Critical limits:** SYNC6 timing calibration measures the **incoming IWM write stream relative to the STM32 timer**. It cannot by itself prove RD voltage levels, pulse width/polarity, tri-state behavior, write-gate setup/hold, oscillator stability or every host electrical tolerance. These require **design validation**, but no user-facing manual oscilloscope calibration should be necessary once hardware is validated. The initial format must function on the nominal timing baseline to obtain the measurement.

**[DECISION]** Store timing calibration in **global cartridge metadata**, distinct from the raw disk image and track directory. The external programmer must preserve it when changing a disk image. **[OPEN]** Calibration portability between different emulator units and host machines, oscillator drift and a fallback/un-calibrated policy require explicit rules.

### 11.4 Logical track descriptors, without a motor-zone model

**[DECISION]** The STM32's primary storage/playback abstraction is **one logical track/side**:

~~~text
LogicalTrack:
    track_number             (0–79 for standard 400K/800K)
    side                     (0 or 1 where applicable)
    sector_count             (standard values 12/11/10/9/8)
    raw_data_offset          (address inside SPI-flash data region)
    raw_data_allocation      (capacity in bytes)
    raw_data_used_bytes      (bytes containing packed bits)
    valid_bit_length         (number of bit cells; need not be multiple of 8)
    track_format_and_flags   (valid, formatted, canonical, raw-only, etc.)
    optional_sector_map      (verified field/bit positions)
    integrity_information    (format and algorithm to be decided)
~~~

**[DECISION]** The STM32 does **not** implement physical changes in track circumference, magnetic density, spindle acceleration or a motor-speed control algorithm. It consumes the **sector count and valid bit length** specified by the logical track descriptor and emits that bitstream with the **one global calibrated cell period**.

For the normal Macintosh format, physical drive geometry explains why different tracks contain 12, 11, 10, 9 or 8 sectors, but the **media descriptor is the firmware contract**. A standard image-conversion tool may use the conventional 80-track/side mapping to create those descriptors; the STM32 read engine should not need to infer sector count from a five-zone motor model.

**[DECISION]** Distinguish **valid bits** from **flash allocation size** and preserve an explicit wrap point. Do not assume the last packed byte is fully valid. Do not insert fictitious index markers or reset the stream on every status read.

**[OPEN]** The exact track bit lengths, seam/gap policy, sector-map format and limits for nonstandard layouts remain to be established. A fresh format and a later sector overwrite can leave different splice bits.

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

**[ROM]** Ordinary writes are **sector/data-field writes**, not whole-track flushes from a Macintosh dirty track cache. The ROM reads address fields, identifies the requested sector and calls `Sony_WrData` on the matching data field. Its track cache is principally a **read cache**; normal writes invalidate cached sectors. **Formatting** uses a different path, `SONY_DOFORMAT`, to transmit a whole track.

**[DECISION]** The STM32 maintains a virtual position in the selected track's continuously advancing, cyclic read stream. For standard tracks, a validated address/sector-position map can associate the host's write-gate event with the sector just identified from the STM32's own outgoing stream.

**[DECISION — architecture, not detailed timing]** Use RAM buffering so captured WR transitions are **not** written directly to SPI flash in a time-critical loop. A useful initial model separates a current **complete track bit buffer** from a **sector-sized incoming write-capture buffer**. The STM32 captures raw pulses/bits, aligns them with the virtual track phase and updates only the written region; unaffected sectors remain intact. The format path must also accept extended whole-track writes.

**[OPEN — Part 4/§12]** The exact read-to-write handoff, `/WRREQ` gate qualification, first/last WR edge, write-splice alignment, asynchronous phase or clock differences, gap overwrite length, handling variable-length writes and recovery from underrun **are not yet settled**. A write-gate edge alone does **not** prove a byte-aligned insertion address. No promise of a particular merge algorithm is made in this part.

**[DECISION]** RAM cache is **not** durability. A Macintosh may finish a sector write and power off or eject without changing tracks; therefore **committing only on track changes is insufficient**. Actual flash commit/journal, atomicity, timing and safe removal remain open for Part 4 and the future storage documentation.

### 11.7 SPI-flash layout and centralized metadata

**[DECISION]** Preferred removable media: **16 Mbit = 2 MiB = 2,097,152 bytes** SPI NOR flash in a 3D-printed floppy-shaped cartridge. The capacity is the current design target, not proof that a particular flash chip, erase scheme or allocation is approved. The STM32 interfaces to the cartridge's electrical contacts and stores **raw GCR plus metadata**.

**[DECISION]** Separate **global media/calibration metadata**, **per-track metadata** and **raw-GCR track data**. Every track descriptor carries an **offset and explicit length** pointing into the flash data area. The track data is **not** required to sit immediately after its descriptor; data blocks may be variable length and relocated without redefining logical track IDs.

**[PROPOSED]** Keep all track descriptors together in a **central directory** and place packed track streams in a separate data region:

~~~text
2 MiB REMOVABLE SPI NOR FLASH (conceptual; no offsets assigned)

+------------------------------------------------+
| Superblock / global media identity / format    |
| Global GCR timing calibration (one per disk)   |
| Version, directory pointers, integrity fields  |
+------------------------------------------------+
| Central track directory: track 0/side 0 ...     |
|  track 79/side 1; generous reserved fields      |
+------------------------------------------------+
| Optional alternate metadata directory / log    |
| (recovery / metadata upgrades)                 |
+------------------------------------------------+
| Raw-GCR packed-bit blocks, addressed by offset |
| and length from each track descriptor           |
+------------------------------------------------+
| Available blocks / future allocation / journal |
+------------------------------------------------+
~~~

**[DECISION]** Reserve **substantial expansion room** within and/or between metadata records, so newer firmware/software can add fields without moving raw-GCR data. The exact record width, directory placement, reserve sizes, number of metadata copies and flash-sector boundaries are **[PROPOSED]/[OPEN]**.

**[PROPOSED — illustrative only]** 160 track/side records × 512 bytes = **80 KiB** for one fixed-width directory; two such directories would consume **160 KiB**. This is an example of generous extensibility, **not a committed on-flash ABI**. Capacity must be checked against worst-case raw-track lengths, spare/erase units, flash updates and wear.

### 11.8 Versioning, forward extension and lossless metadata upgrades

**[DECISION]** A previously formatted cartridge should be **upgradable to newer metadata versions without losing its raw-GCR disk contents**. The external utility and firmware must not silently reinterpret or overwrite an unknown old layout. Separate versioned metadata from raw data and preserve offsets/lengths across compatible migrations.

**[PROPOSED]** Give the cartridge format a **versioned superblock** and each directory/record a recognized schema version or length. Reserve ample field space and provide checksums/CRC and an **alternate recoverable metadata generation**. A safe metadata-only migration would:

1. Read and validate old header, directory and referenced raw-track extents.
2. Build the new directory/schema in unused or alternate metadata storage, keeping raw-track bytes and bit lengths intact.
3. Verify the new metadata and raw-data references.
4. Atomically activate the new directory generation using a flash-safe method.
5. Reclaim obsolete metadata only after the new version is durable.

The precise flash atomic-update protocol is **[OPEN]**: SPI NOR has erase granularity, 1→0 programming constraints, limited endurance and the risk of power interruption. These issues require a real design, not just a spare directory copy.

**[PROPOSED]** Keep distinct compatibility versions for **container/metadata schema**, **packed-bit track representation**, and **calibration encoding**. Extending metadata should normally leave raw GCR untouched; changing the representation itself may require data conversion.

### 11.9 Cartridge formatting, external programming and reuse

**[DECISION]** The reference user workflow is:

~~~text
1. Insert blank SPI-flash floppy cartridge into emulator.
2. Macintosh Plus formats it through original ROM + IWM.
3. STM32 captures the format stream and learns global SYNC6 timing;
   preserves correct formatted-track state and metadata.
4. Remove cartridge using a safe eject/commit sequence.
5. Insert cartridge into a separate host-attached flash programmer.
6. External program imports a compatible Macintosh disk image,
   builds complete 6-and-2 raw-GCR track streams with correct
   sync, marks, sectors, checksums and track metadata,
   and PRESERVES the cartridge's global timing calibration.
7. Return cartridge to Macintosh; STM32 replays the raw-GCR bits
   with the stored global timing while providing correct status/TACH.
~~~

The external image-preparation program does **not** need to generate real-time serial electrical transitions. It creates valid **bit order, field structure, track length and integrity information**. Standard `SYNC6` can be generated internally in the emulator and/or by the offline conversion tool, as long as the resulting mapped standard stream is correct.

**[DECISION]** **Formatting is an initial calibration/initialization pathway, not a requirement before every image import.** Once calibrated, the cartridge can be reprogrammed repeatedly while retaining its calibration. **[OPEN]** An uncalibrated cartridge's behavior, reusable device-level profiles and transferability between Macintosh hosts require explicit policy.

**[OPEN]** Image sources can be decoded 400K/800K disk images, tag-bearing images or raw-track/flux-derived formats. They do not all preserve original tags, write splices, nonstandard tracks or copy protection; conversion must report limitations and must not claim bit-perfect reproduction from information the source image does not contain.

### 11.10 Constraints and interfaces for subsequent parts

**Part 1 / §§1–9:** Retains connector pin mapping, status/control semantics, track stepping, insertion/ejection, readiness, hardware 800K TACH evidence and the ROM's speed-check behavior. The **sector-count → TACH table** in this part is a *firmware contract*; physical zones remain useful as historical source context only.

**Part 2 / §10:** Retains the electrical RD waveform, bitstream clock, complete freshly formatted track reference, sync-pattern bit layout, GCR sector marks and cyclic playback. Raw-GCR flash is the source of ordinary read data. Canonical sync construction is a bounded track-preparation function, not an arbitrary alteration of the read stream.

**Part 4 / §12 — serial GCR writing (planned):** Specify gate/WR edges, timer capture, bit alignment and splices, normal sector writes vs whole-track format, shared RAM buffer state and durable flash commits. It must honor the global calibration and per-track bit lengths described here.

**Later docs, §§13–18 (planned):** §13 formatting/400K details; §14 errors; §15 electrical timing; §16 STM32 firmware/peripherals; §17 verification; §18 ongoing open questions/decision log. Proposed **Document 50 onward** may expand cartridge allocation, calibration, programmer software and image conversion, but cannot silently contradict this shared architectural baseline.

### 11.11 Decisions, proposals and open questions

**Settled [DECISION]:**

- Original Macintosh ROM performs 6-and-2 conversion; IWM performs serial read/write assembly; STM32 handles timed Sony-drive-compatible pulses and raw-bit capture/replay.
- Removable **16-Mbit (2-MiB target) SPI NOR** contains packed raw-GCR bitstreams; preserve the **valid bit length** and exact ordinary track playback.
- Internally generate the canonical `SYNC6` pattern for **mapped standard** sync regions; preserve unknown/nonstandard raw data unchanged.
- **One global** GCR bit-cell rate/calibration, common to all track/sides; project nominal **489,600 bit/s**, bench confirmation pending.
- Format-time repeated `SYNC6` is the preferred automatic timing reference, with no per-zone calibration.
- Tracks are described **logically** by track/side, sector count, bit length and flash data offset/length; no spinning-motor simulator or physical-zone mechanism model.
- Host-visible **TACH is required** even for an 800K drive; a sector-count-derived frequency table supplies the appropriate status waveform, independent of GCR calibration.
- ROM ordinary writes affect **individual data fields**; track formatting is a separate whole-track write. STM32 buffers changes in RAM and eventually persists them safely.
- Centralizable metadata, reserved upgrade space and future **metadata migration without erasing raw-GCR disk contents** are required design goals.
- Support Macintosh format/calibration → external disk-image program → Macintosh read/write workflow.

**Proposed, not frozen:**

- Central fixed-width track directory with generous spare bytes; redundant directory generations/superblock and CRC.
- Whole-track RAM read buffer plus a separate incoming sector-sized capture buffer.
- Internal sync replacement based on validated track-region maps and optional standard-track normalization mode.
- One versioned on-flash container separating metadata schema, raw-bit representation and calibration encoding.
- Stable STM32 external oscillator and hardware timer/input-capture for SYNC6 characterization.

**Still open (do not infer settled implementation details):**

- Specific STM32, SPI NOR part, power/contacts and on-flash addresses/record sizes.
- Exactly how the **first usable format** is supported before calibration; calibration validity, host-to-host portability and frequency drift.
- Exact complete-track seam, `GapSync` size, bit alignment and metadata needed to preserve physical-sector write splices.
- Precise WR and `/WRREQ` handling, write gate to virtual-bit-position relationship, formatting write capture, underrun behavior.
- Flash sector/program/erase granularity, journaling/atomicity, wear, durability, media removal and rollback after interruption.
- Image format import coverage, preservation of tags/nonstandard tracks, file-to-track serialization and validation fixtures.
- Practical coherence checks between track bit length, TACH status and ROM speed expectations; electrical pulse timing and bench qualification.

**Document history**

- **2026-11-09:** Created **06 — Part 3 / §11** to consolidate agreed media, SYNC6 calibration, logical track, TACH and external programmer architecture. Renumbered planned **serial writing to Part 4 / §12**, and deferred later protocol sections to **§§13–18**. No binary flash layout or write-timing algorithm is implied by this documentation reorganization.
