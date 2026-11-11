# Sony MFD-51W Drive Interface Protocol — Part 6: Error Handling and Standard-Media Limits

**Document:** 06 — Part 6  
**Protocol section:** 14 — Error handling and unexpected sequences  
**Status:** Agreed first-release standard-media scope; error-handling requirements with unresolved hardware/flash mechanisms marked **[OPEN]**  
**Target:** Macintosh Plus M0001AP, original ROM .Sony driver and IWM, Sony MFD-51W-compatible interface  
**Date:** 9 November 2026

> **Document sequence:** [Part 1](06-part-1-drive-interface.md) covers drive pins, logical control/status, media, write protection and eject. [Part 2](06-part-2-serial-gcr-read.md) defines RD waveforms. [Part 3](06-part-3-media-architecture.md) defines 735-byte structured-GCR sector records, global calibration, cache and cartridge data. [Part 4](06-part-4-serial-gcr-write.md) defines qualified WR capture, formatting and persistence. [Part 5](06-part-5-format-geometry-400k.md) fixes the standard 400K/800K format and logical circular-track reconstruction. **Part 6 governs unusual operations and failures on otherwise standard media.**

**Evidence notation:** **[ROM]** original Plus driver; **[SONY]** published host interface; **[DECISION]** agreed project requirement; **[PROPOSED]** possible implementation; **[OPEN]** further hardware/firmware design or measurement. Do not invent Sony interface signals or assume the MCU can directly return a Macintosh software error code.

## Contents

- [14.1 Scope and governing invariants](#141-scope-and-governing-invariants)
- [14.2 Standard images and external encoding](#142-standard-images-and-external-encoding)
- [14.3 Nonstandard tracks outside the first release](#143-nonstandard-tracks-outside-the-first-release)
- [14.4 ROM-visible errors and emulator limitations](#144-rom-visible-errors-and-emulator-limitations)
- [14.5 No media, blank media and damaged media](#145-no-media-blank-media-and-damaged-media)
- [14.6 Write protection and invalid write requests](#146-write-protection-and-invalid-write-requests)
- [14.7 Interrupted ordinary sector writing](#147-interrupted-ordinary-sector-writing)
- [14.8 Interrupted track formatting and retries](#148-interrupted-track-formatting-and-retries)
- [14.9 Repeated sector writes and dirty-cache coherence](#149-repeated-sector-writes-and-dirty-cache-coherence)
- [14.10 Three-buffer exhaustion and flash failures](#1410-three-buffer-exhaustion-and-flash-failures)
- [14.11 Unexpected cartridge removal and safe eject](#1411-unexpected-cartridge-removal-and-safe-eject)
- [14.12 Unusual control and selection sequences](#1412-unusual-control-and-selection-sequences)
- [14.13 Fault handling state model](#1413-fault-handling-state-model)
- [14.14 Verification matrix](#1414-verification-matrix)
- [14.15 Decisions and engineering gates](#1415-decisions-and-engineering-gates)

---

## 14. Error handling and unexpected sequences

### 14.1 Scope and governing invariants

**[DECISION]** The first firmware supports **standard Macintosh 400K and 800K disks only**, with standard 12/11/10/9/8-sector track classes, one or two formatted sides, valid unique sector IDs, normal headers/checksums, standard 6-and-2 codewords and ROM-compatible SYNC6/gaps/track timing. The emulator must handle all normal ROM drive operations **and their error/interruption paths**, but not arbitrary exotic magnetic recordings.

**Safety invariants:**

1. **Do not invent success:** an invalid write, failed flash operation or unknown track must not be silently converted into successfully stored data.
2. **Protect unrelated data:** an ordinary incomplete write must leave other sectors unchanged; an interrupted format may invalidate its **target track** but must not damage other tracks, cartridge directory or calibration.
3. **Retain truthful status:** no simulated SPI-flash ACK or generic error signal exists on the Sony connector.
4. **RAM is not durable:** dirty data remains protected until recoverable/committed; software eject must not release uncommitted accepted writes.
5. **No mechanical simulation:** preserve specified host-visible electrical behaviour without inventing seek delays, spin-up, rotational latency or acoustic effects.
6. **Reject unsupported representation safely:** do not reinterpret an unknown layout as writable structured sectors.

### 14.2 Standard images and external encoding

**[DECISION]** External cartridge programming imports **standard logical-sector disk images** and performs GCR encoding offline. A plain Macintosh 800K image typically contains **1,600 decoded 512-byte sectors (819,200 bytes)**, not pre-recorded 6-and-2 GCR gaps, SYNC6 and headers. Similarly, a plain 400K image contains **800 decoded 512-byte sectors (409,600 bytes)**.

| Source | Typical contents | Required importer behavior |
|---|---|---|
| Plain 400K image | 800 user sectors, no recorded physical GCR | Generate normal side-0 format, encoded headers/checksums, 735-byte records |
| Plain 800K image | 1,600 user sectors, no recorded physical GCR | Generate normal two-sided format and encoded records |
| Supported tag-bearing image (e.g. Disk Copy 4.2) | User sectors and possibly per-sector 12-byte tags | Preserve tags when provided; validate container and encode standard tracks |
| Invalid/unknown image | Unsupported geometry, container or damaged data | Reject the import; never partially publish it |

**[DECISION]** Image import supplies the agreed **32-byte metadata containing preserved/generated encoded address and data headers**, and **703 GCR payload/checksum codewords** per sector (699 tag/user-data codewords plus 4 checksums). If input lacks **12-byte sector tags**, generate them only under an **explicit documented tag-synthesis policy**; do not claim to preserve tag information absent from the source. Use normal ROM 2:1 interleave unless a supported source explicitly provides another valid physical sector permutation.

**[DECISION]** The external programmer verifies image size/format, applicable container integrity, physical capacity and on-flash directory compatibility **before** replacing a cartridge image. Importer-specific parsing/encoder design is a separate future workstream.

**[REFERENCE]** [CiderPress2 DiskCopy format notes](https://ciderpress2.com/formatdoc/DiskCopy-notes.html) describe decoded sectors and optional tags, as distinct from a bit-level flux/track recording.

### 14.3 Nonstandard tracks outside the first release

**[DECISION]** The first firmware **does not need to support** deliberately malformed sector headers, duplicate/missing IDs as a valid formatted track, copy-protection tracks, weak/random bits, unusual GCR speed tricks, arbitrary custom gaps/synchronization, raw flux imports or arbitrary bit-splice RAW_TRACK writing.

**[DECISION]** The centralized directory may retain a **reserved RAW_TRACK representation ID** for future expansion, but **the first firmware is not required to implement RAW_TRACK reading or writing**. A cartridge that requests an unsupported representation must be rejected/marked unsupported, not silently normalized or erased. Do not interpret a reserved type field as a commitment to a second engine.

**[DECISION]** A valid **permutation of normal unique Macintosh sector IDs** remains representable as STRUCTURED_GCR. The Plus ROM formatter produces 2:1 interleave; its ordinary read/write code examines address sector numbers. This harmless flexibility does not imply arbitrary copy-protection support.

### 14.4 ROM-visible errors and emulator limitations

**[ROM/SONY]** Macintosh error codes such as **offLinErr, noDriveErr, wPrErr, wrUnderrun, fmt1Err, fmt2Err and verErr** originate in ROM software and its observations of actual drive signals and GCR data. The Sony connector has **no general SPI-flash error register, sector-write ACK or host write-pause/flow-control signal**. The STM32 cannot directly “send” these software errors.

**[DECISION]** Provide authentic RD/status/protection behaviour, and allow the ROM to discover missing address/data, write protection, format verification failure or not-ready conditions. No invented command, extra status bit or fabricated successful readback is permitted.

**[OPEN]** Define the final host-visible policy for a **present cartridge with unrecoverable SPI fault**: present-but-not-ready versus logically absent. Never automatically pretend damaged backing storage is healthy blank media.

Sources: [Plus ROM ordinary-sector search](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L888-L1012), [format/gap verification](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2883-L2955), [full-disk Verify](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L3132-L3245).

### 14.5 No media, blank media and damaged media

**[DECISION]** Distinguish cartridge presence, cartridge qualification, track formatting, sector validity and storage health.

| Condition | Behaviour |
|---|---|
| No cartridge / unqualified contacts | Do not read/write stale content; indicate absent/not-ready via the standard Sony statuses |
| Healthy blank cartridge | Present and write-eligible for original-ROM formatting; **do not synthesize already-formatted address fields** |
| Standard formatted cartridge | Reconstruct the selected track and present valid cyclic GCR/RD while ready |
| Corrupt or incomplete sector/track | Never silently regenerate a valid sector from unknown/truncated bytes; allow genuine read/Verify failure |
| Invalid flash directory or unsupported type | Do not present a fabricated healthy formatted disk; enter qualification/fault policy |
| Cartridge removed during an operation | Stop old media access and invalidate old cartridge-specific cache/flash work |

**[DECISION]** Blank and faulty are not interchangeable. First-format bootstrapping must work with a blank healthy cartridge using provisional global timing and known standard geometry. A read of unformatted media must not claim good sector data; its specific RD/READY behaviour requires testing with the Plus ROM.

### 14.6 Write protection and invalid write requests

**[DECISION]** The STM32 checks actual media permission **for each write**, even if the Macintosh ROM has cached an earlier writable status. Protected media can still be read when valid. No GCR capture may modify RAM/flash data or schedule erase on a protected cartridge.

A write must be rejected/ignored as **unqualified** if drive enable/gate conditions are wrong, the cartridge is absent or removed, the requested side is unsupported, selection changed incompatibly, or the current cartridge generation no longer matches the original target. Do not confuse a status-source selection or motor-off with a write or eject command.

**[OPEN]** The precise ROM-visible outcome of a write-protect transition during a mounted session depends on ROM caching and Sony pin timing. Re-presentation through logical eject/reinsertion is a conservative policy, to be verified rather than assumed.

### 14.7 Interrupted ordinary sector writing

**[ROM]** Normal Sony_WrData sends one SYNC6, the **D5 AA AD** data header plus encoded sector number, **703 encoded data/checksum bytes** and **DE AA FF FF**. The existing address field is not rewritten.

**[DECISION]** Capture WR **both-edge transitions** within a qualified gate, identify a candidate target from the last valid address field served on RD and validate it against the incoming sector ID. Accept and publish the new sector **only after** the complete recognized field and relevant checksums are validated.

An ordinary write is incomplete/invalid when: gate terminates prematurely, WR transitions are missing/undecodable, prologue/end markers fail, codewords/checksums are invalid, DMA capture overflows, sector address is ambiguous/mismatched, or protection/media identity changes before acceptance.

**[DECISION]** Retain the prior valid sector on rejection. Preserve other sectors, address headers, entire other tracks and directory. Never commit partial GCR data or invent missing checksums. **[OPEN]** The Macintosh may have ended its electrical write without knowing that an internal validation failed; there is no per-write ACK for deterministic instantaneous notification.

### 14.8 Interrupted track formatting and retries

**[ROM]** Formatting emits **200 initial SYNC6 groups**, each sector header and 703-byte GCR field, then checks the next address for **sector 0** and evaluates the gap. The ROM can adjust **GapSync** and retry the entire track.

**[DECISION]** Detecting **50+ consecutive SYNC6 groups** inside a single qualified gate is an early **candidate-format** signal, never sufficient proof of a complete format. Validate all expected unique sector IDs, headers, fields, side/track/format, codewords/checksums and the reconstructed standard cyclic track. Publish for immediate genuine ROM sector-0 readback when valid; flash persistence may continue asynchronously.

**[DECISION]** A format retry **replaces**, rather than appends to, the prior candidate. Formatting intentionally destroys existing target-track information: an interrupted format may leave **that target track** invalid/unformatted. It must not corrupt **unrelated tracks, global calibration or the cartridge directory**. Keep the ROM-compatible gap/seam requirements settled in Section 13; physical bench testing remains necessary.

**[OPEN]** Atomicity/recovery of an early isolated 4-KiB erase and track metadata activation must be designed for real power-cut behavior.

### 14.9 Repeated sector writes and dirty-cache coherence

**[DECISION]** Multiple ordinary writes to the **same standard sector or same cached track** must be supported. Replace only the validated data-field header/703 codewords, leave the address header and other records untouched, publish a coherent new RAM generation and mark the track dirty.

**[DECISION]** Read-after-write uses the **latest accepted RAM generation**, not an older flash copy. The independent SPI worker should **coalesce** writes on a dirty track when possible, and must never persist an older generation over newer accepted data. Track changes alone are not a sufficient persistence guarantee.

### 14.10 Three-buffer exhaustion and flash failures

**[DECISION]** Use **at least three independently managed complete-track buffers of at least 12 KiB each**, separate WR edge-capture rings and a nonblocking SPI flash worker. Buffer ownership is explicit: the sole dirty or actively streamed generation cannot be reused until safe.

**[DECISION]** Detect cache-buffer exhaustion, capture overrun, pending flash backlog, erase/program timeout, verification mismatch, bad directory CRC/metadata and cartridge disconnection. Flash activity must not interrupt deterministic RD output or WR capture.

**[IMPORTANT LIMIT]** Three buffers are a **minimum resource provision**, not proof of unlimited write throughput. The interface lacks a general write-pause/ACK mechanism. With all buffers pinned and dirty, another incoming host write may not be transparently serviceable. The implementation must **demonstrate normal continuous ROM workloads**, define a deterministic fault/overflow response, and **never silently discard data while claiming success**.

**[DECISION]** A dirty cache generation is durable only when the flash transaction and associated recoverable directory state have been verified. After a flash failure, preserve valid RAM while power exists, avoid further unsafe writes and transition to a documented write-disabled/fault state.

**[OPEN]** Specific journal/COW/in-place-erase policy, wear/endurance, backlog bounds, metadata commit atomicity and crash recovery are Section 16 implementation work, with Section 17 fault-injection tests.

### 14.11 Unexpected cartridge removal and safe eject

**[DECISION]** Software **EJECT** is not physical withdrawal, drive deselection or MOTOR OFF. Before a successful logical eject/release, accepted dirty writes must be **durable or demonstrably recoverable**. Block new writes during the eject sequence; do not falsely announce success or reannounce the same physically seated cartridge.

**[DECISION]** Unexpected cartridge removal requires immediate guarding of SPI contacts, suppression of stale RD, qualified not-ready/absent status and cancellation of cache/flash work belonging to the removed cartridge. A newly inserted cartridge must be identified by a new generation and must never receive an old buffer's delayed write.

**[DECISION]** Power loss can destroy volatile RAM updates. Use recoverable persistent metadata and a validated transaction strategy; **do not claim SRAM survives power loss**. Sudden power/removal may lose uncommitted changes unless hardware/energy protection prevents it, so the complete durable-write/eject guarantee remains an explicit engineering gate.

**[OPEN]** Contact sequence, release interlock, power-fail warning/hold-up energy, safe flash programming duration and the ROM eject time window require hardware design and measurement.

### 14.12 Unusual control and selection sequences

**[DECISION]** Even with only standard formatted media, handle:

- Multiple STEP pulses, logical track-zero limits and attempts past track 79 without memory indexing or cross-track writes.
- Head/side changes or deselection during a write gate without redirecting the remainder to another track; latch and validate the original target.
- Repeated status/TACH selections without restarting the cyclic read phase unnecessarily.
- Motor OFF and host deselection without pretending the cartridge was ejected or successfully flushed.
- A standard **400K single-sided image in an 800K-capable drive** without fabricating side-1 recorded sectors.
- Repeated format of the same track and immediate format check, valid retries, repeated ordinary-sector writes and ROM read/Verify retries.

**[OPEN]** Exact gate/selection races and host-visible timeout/error interpretation must be verified against the real Plus using [Part 7 / §15 electrical requirements](06-part-7-electrical-timing.md) and future Section 17 validation.

### 14.13 Fault handling state model

**[PROPOSED]** Keep **media qualification**, **active read/cache**, **write capture**, **flash persistence** and **safe eject** as distinct state machines or explicit orthogonal state components:

~~~text
MEDIA: ABSENT | QUALIFYING | BLANK | STANDARD_VALID | FAULT | EJECTED
TRACK: UNFORMATTED | CLEAN | CAPTURING | DIRTY | FLUSHING | INVALID
WR:    IDLE | SECTOR_CANDIDATE | FORMAT_CANDIDATE | VALIDATING | REJECTED
FLASH: IDLE | PREPARING | ERASING | PROGRAMMING | VERIFYING | COMMITTING | FAILED
EJECT: NONE | BLOCK_NEW_WRITES | FLUSH_PENDING | RELEASE_SAFE | FAULT
~~~

**[DECISION]** Each buffered write and flash operation is tagged by **cartridge identity, track/side, generation and validity**, so errors cannot make a previous cartridge's data appear on the next one. These labels are internal: no extra Sony pins.

### 14.14 Verification matrix

| Scenario | Required result |
|---|---|
| No cartridge | No stale read; absent/not-ready and no write |
| Blank healthy cartridge | Standard ROM format possible without invented pre-existing sectors |
| Protected cartridge write | No cache/flash modification despite any host-cached permission |
| Interrupted ordinary sector write | Previous sector and all other sectors remain safe; no partial acceptance |
| Interrupted full-track format | Target may be invalid; unrelated track/directory/calibration intact |
| ROM format retry with changed GapSync | Replace same target; genuine immediate sector-0 readback |
| Repeated writes to one sector | Most recent accepted GCR field read back; safe write coalescing |
| Unexpected removal | Guard SPI, suppress stale RD, no delayed write to another cartridge |
| SPI erase/program/verify failure | Detect; no false-clean cache generation or silent corruption |
| All three track buffers pinned | Detect and report internally; no silently discarded accepted write |
| Dirty cache plus software eject | Block further writes; release only after durable/recoverable commit |
| Power loss during transaction | Recover committed directory/tracks; no corruption of other data |
| Nonstandard/RAW_TRACK input | Safely reject unsupported representation; no normalization |
| 400K single-sided image | Correct side-0 operation, no synthetic side-1 sector |
| Valid nondefault physical-sector permutation | Preserve unique IDs/order within standard structured format |
| Repeated STEP/status/enable changes | No out-of-bounds or cross-track data corruption |

**[DECISION]** These are required **acceptance tests**, not a claim that their firmware algorithms are already complete. Their fault-injection fixtures belong in Part 9 / Section 17.

### 14.15 Decisions and engineering gates

**Settled [DECISION]:**

- Initial release supports **standard 400K/800K Macintosh structured-GCR**. Arbitrary RAW_TRACK playback/writes and nonstandard/copy-protected recording formats are **out of scope**.
- An existing/reserved RAW_TRACK representation ID remains a **future format extension**, not a first-release requirement.
- External programmer converts logical-sector disk images to encoded GCR; original sector tags absent from source require an explicit creation policy.
- **All normal operational failures still require safe handling:** missing/blank media; protection; incomplete sector write; interrupted format; GapSync retry; repeated writes; physical removal; SPI flash failure; three-buffer backlog; dirty eject; power cut; unexpected selector/step sequences.
- Standard geometry, GCR payload/header structure, circular track and global calibration from Parts 2–5 remain unchanged.
- No fabricated ROM error channel or false write/eject success. Protect unaffected sectors/tracks and cartridge metadata.

**[OPEN — implementation/validation]** Hardware-specific WR/RD timing and fault visibility; precise 32-byte record ABI and image tag policy; flash transaction journal/commit/wear; cache peak occupancy and emergency handling; power integrity and safe physical eject; ROM loopback/fault tests.

**Document history**

- **2026-11-09:** Created Part 6 / Section 14. Clarified standard-media-only first-release scope; RAW_TRACK left reserved but unsupported initially. Covered the approved unusual **operational** sequences and error cases without promising arbitrary exotic magnetic recording.
