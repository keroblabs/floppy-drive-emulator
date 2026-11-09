# Sony MFD-51W Drive Interface Protocol — Part 2: Serial GCR Reading

**Document:** 06 — Part 2  
**Protocol section:** **10 — Serial GCR disk-data reading**  
**Status:** Detailed research specification; electrical pulse generation, phase/format edge cases and STM32 timing architecture require bench validation  
**Target:** Macintosh Plus M0001AP with its original IWM and ROM `.Sony` driver  
**Mechanism:** Sony MFD-51W-compatible internal 800K drive, with connector pins 9 and 20 electrically disconnected  
**Preferred implementation:** STM32 (specific part/peripherals not yet selected)  
**Date:** 9 November 2026

> **Document split:** This continues [06 — Part 1: Host interface and drive operations, Sections 1–9](06-part-1-drive-interface.md). **This Part 2 contains Section 10 only**. The shared decisions on raw-GCR media, SYNC6 calibration, logical track metadata, TACH and cartridge programming are consolidated in **[06 — Part 3 / Section 11](06-part-3-media-architecture.md)**. Detailed **serial writing is planned for 06 — Part 4 / Section 12**; later sections are **13–18**.
>
> **Design rule:** Emulate the *electrical and logical* read interface, **not** spinning hardware, sounds, seek time or motor inertia. Yet the IWM still expects correctly timed **serial pulse patterns** and recognizes recorded fields as they arrive. Do not mistake “no mechanical rotational latency” for permission to output whole bytes instantly or substitute a static status value for disk data.

## Contents

- [10. Serial GCR disk-data reading](#10-operation-serial-gcr-disk-data-reading)
  - [10.1 Scope and separation from status reads](#101-scope-and-separation-from-status-reads)
  - [10.2 Physical signal selection and read enable](#102-physical-signal-selection-and-read-enable)
  - [10.3 How the IWM receives a GCR byte](#103-how-the-iwm-receives-a-gcr-byte)
  - [10.4 Sony RD waveform and source timing](#104-sony-rd-waveform-and-source-timing)
  - [10.5 GCR encoding and self-synchronisation](#105-gcr-encoding-and-self-synchronisation)
  - [10.6 Full sector structure](#106-full-sector-structure)
  - [10.7 The five zones and track wrapping](#107-the-five-zones-and-track-wrapping)
  - [10.8 Macintosh Plus ROM routines and error cases](#108-macintosh-plus-rom-routines-and-error-cases)
  - [10.9 Streaming model, transitions and storage interaction](#109-streaming-model-transitions-and-storage-interaction)
  - [10.10 Potential STM32 timing architecture](#1010-potential-stm32-timing-architecture)
  - [10.11 Verification and performance acceptance](#1011-verification-and-performance-acceptance)
  - [10.12 Open questions and design decisions](#1012-open-questions-and-design-decisions)

---

## 10. Operation: serial GCR disk-data reading

**Scope:** Reproduce what the Macintosh Plus's **IWM input** receives from a Sony 800K mechanism when the head-0 or head-1 read source is selected onto RD. This section specifies a continuous *host-visible* read stream for 400K and 800K GCR media, not the physical magnetic read head or disk controller itself.

**Evidence labels:** **[SONY]** original Sony/Apple drive electrical specification; **[IWM]** Apple IWM/SWIM architecture description; **[ROM]** observed Macintosh Plus `.Sony` driver behaviour; **[DECISION]** proposed emulator contract; **[OPEN]** item requiring further evidence or test.

### 10.1 Scope and separation from status reads

The common internal 20-pin drive connection carries **only one RD output** (pin **16**) for both addressed status and serial read data. It is the **same physical line** but not the same operating mode:

| Operation | Sony selected RD source | Macintosh IWM mode | What the 68000 sees |
|---|---|---|---|
| **Status reading** (Part 1 `4) | CSTIN, WRTPRT, READY, TK0, etc. | `Q7=0, Q6=1` | Sense/status value in bit 7 |
| **Disk-data reading** (this Part 2) | **RDDATA0** or **RDDATA1** | `Q7=0, Q6=0` | IWM **assembled GCR nibble/byte**, with bit 7 used as the byte-complete indication |
| **Control command** (Part 1 `6) | RD still carries whichever selector is active; write is **LSTRB** | IWM PH3 controls strobe | Drive accepts control on qualified LSTRB rising edge |

The STM32 **does not receive Q6/Q7 as connector signals**. It must present the correct time-varying RD output based on **CA0–CA2, SEL, /ENBL**, logical motor/readiness and its stream state; the IWM subsequently converts pulses into bytes in its own circuitry.

### 10.2 Physical signal selection and read enable

**[SONY]** The two data-source selector states are:

| Source | CA2 | CA1 | CA0 | SEL | Project address | ROM address/symbol |
|---|---:|---:|---:|---:|---|---|
| **RDDATA0 / side 0 / head 0** | 1 | 0 | 0 | 0 | **`0x8`** | **`1` / `RdDtaAdr`** |
| **RDDATA1 / side 1 / head 1** | 1 | 0 | 0 | 1 | **`0x9`** | **`3` / `RdDta1Adr`** |

**[ROM]** [`Sony_RdAddrSetup`, lines 2094–2106](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2094-L2106) selects `RdDtaAdr` or `RdDta1Adr` according to bit 11 of the ROM's side/track variable; the low-level selector routine sets CA0–CA2 through the IWM and **SEL through VIA port A bit 5**.

**[SONY]** **/ENBL.INT=0** is required to connect the selected drive output. When **/ENBL.INT=1**, RD is **high impedance** (the specified output-disable time is at most **0.5 µs**; see Part 1 and `10.4). No other drive-output GPIO should source RD.

**[DECISION]** Switching sides changes which track-side stream is exposed on RD, **not** the virtual radial track number. If a side change occurs during a read, the stream must be redirected safely to the correct side; the RD mux must not simultaneously drive a static sense level and a GCR stream. The precise phase-coherence requirement between heads remains **[OPEN]**.

**[OPEN]** The Apple 1986 699-0452-A specification mentions gating of RDDATA by PWM on certain external drive-unit configurations (699-0326). **Do not apply that external-unit behaviour to our internal MFD-51W**, for which pin **20 remains NC**. Original MFD-51W and MFD-51W-03 harness compatibility still requires final electrical validation.

### 10.3 How the IWM receives a GCR byte

**[IWM]** Apple's *SWIM Chip User's Reference* `GCR Encoding and `Using the IWM Hardware explains the data path. The IWM is the **serial-to-parallel decoder** for recorded pulses. It derives encoded bits from the time spacing of RD pulses, collects bits into **8-bit GCR code bytes** (“nibbles”), and makes them available in its **read-data register**. We are *not* building an external byte-wide data bus.

**[ROM]** [`Sony_RdAddr`, lines 2113–2149](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2113-L2149) explicitly states the IWM read state (`Q7=Q6=0`, drive enabled). It reads the IWM **`q6L`** location and tests **bit 7**:

~~~asm
; Illustrative extract of the Plus's inner polling logic (Sony_RdAddr)
    MOVE.B  (A4),D5       ; A4 = IWM_Q6L: read assembled nibble
    BPL.B   ...           ; bit 7 clear => no complete nibble yet
    ; bit 7 set => nibble available; process mark/data
~~~

This **bit 7** is a property of the IWM's read-data register, **not an extra RD pin** or STM32 status signal. Correctly timed incoming GCR flux-transition pulses cause the IWM to produce complete high-bit-set code bytes. A loop that reads `q6L` too early sees no new nibble.

**[IWM]** GCR bytes are sent **most-significant bit first** in a continuous serial stream. The IWM recognizes a valid byte boundary from the recorded self-synchronizing pattern and encoding constraints. There is no sector-request command to the Sony mechanism: the ROM waits for recognisable address/data marks in the stream.

### 10.4 Sony RD waveform and source timing

**This is the central real-time constraint of Part 2.** The STM32 must generate short, appropriately spaced **electrical pulses** on RD, rather than merely toggle a logic bit per byte or issue asynchronous GPIO updates whenever software has time.

**[SONY]** Apple 699-0452-A, `2.3.2 (printed sheet 5) specifies **489.6 kbit/s ±0.1%** for the host-to-drive nominal data rate and detected transitions separated by **1.89–6.36 µs**. The **read-data waveform** in `3.4.1.2 (printed sheet **24**, PDF index 23) specifies:

| Parameter | Published value | Practical interpretation |
|---|---|---|
| **RDDATA pulse width T4** | **0.3 µs minimum, 0.8 µs maximum** | The RD pulse *width* is sub-microsecond and must be generated deterministically |
| **Pulse-to-pulse spacing T5** | **2, 4 or 6 µs nominal** | GCR bit pattern controls the interval between pulses |
| **T1** | **0.5 µs maximum** | RD-valid timing with /READY in the drawing; exact edge association must be checked against the figure |
| **T2** | **100 µs maximum** | Data switching/validity during head/SEL transition; exact edge association requires diagram-level validation |
| **T3** | **620 µs maximum** | Data resumption associated with write-gate transition; exact edge association requires diagram-level validation |
| **T6** | **100 µs minimum** | Specified SEL/write-gate separation in diagram; validate exact sequencing when integrating future Part 4 / Section 12 |

**[IWM]** Apple's *SWIM Chip User's Reference* uses **2 µs per IWM GCR bit cell** in its fast-mode explanation. Thus `1` bits generate pulses and `0` bits do not, producing **2, 4 or 6 µs nominal** intervals for valid GCR patterns. A continuously repeating *electrical level* is not the code stream; discrete pulses represent flux transitions.

**[DECISION — project timing baseline]:** Use **489,600 bit/s = 2.042484 µs per bit cell** as the nominal STM32 raw-GCR stream timing, pending a planned logic-analyser check on real hardware. The IWM/SWIM reference's rounded **2 µs** explanatory cell is not the selected implementation value. For example, nominal 1/2/3-cell transition intervals at our selected rate are approximately **2.042/4.085/6.127 µs**; keep the original Apple waveform/tolerance requirements for electrical acceptance.

**[SONY]** Apple `3.4.1.1 (printed sheet **23**, PDF index 22) also gives **≤0.5 µs** for an enabled RD output to reflect an addressed signal change and **≤0.5 µs** to reach high impedance after disable. These are **electrical response requirements**, not permissions for arbitrarily high interrupt latency.

**[DECISION]** The eventual output path must meet pulse *edge placement* and *width* together, worst-case over sustained activity. DMA/timer or other deterministic hardware may be appropriate; an unproven ordinary GPIO polling loop is **not** an approved design.

### 10.5 GCR encoding and self-synchronisation

**[IWM]** In Macintosh GCR a **1 bit produces a pulse**, while a **0 bit produces no pulse** in the corresponding bit cell. Runs of more than **two consecutive zeros** are forbidden in ordinary GCR code bytes because the IWM cannot recover the bit timing reliably over longer intervals. The **high bit must be 1** on normal encoded bytes, providing a recognizable byte boundary.

**[SONY]** Of 256 possible byte values, **64** are used as the six-bit data-code alphabet; **`$D5`** and **`$AA`** are reserved as field-mark bytes. A 6-bit value 0–63 maps through a **64-entry lookup table** to a legal 8-bit on-disk GCR code (for example, value 0→`$96`; value 63→`$FF`). Do *not* simply output raw 512-byte sectors as RD bits.

**[ROM]** The Plus uses [`SONY_DENIBLTBL`, lines 2508–2530](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2508-L2530) for GCR decoding and [`DT_Sony_NiblTbl` around line 2730](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2724-L2738) for encoding.

**Self-sync is mandatory.** Address and data fields are preceded by **special synchronization patterns** that realign the IWM to byte boundaries, regardless of where the host begins reading. The ROM's `SONY_WRDATAMKS` table around [lines 2531–2539](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2531-L2539) lists the six bytes:

~~~text
FF 3F CF F3 FC FF      (special overlapping self-sync pattern)
~~~

The bytes are **not six ordinary independently aligned `$FF` nibbles**: the concatenated bitstream includes the bit-slip/synchronization structure. The IWM must see the correct *serial pattern*; simply concatenating sector records without sync/gap structure is insufficient.

**[DECISION]** Treat the encoded track as a stream of **bit cells** (or an equivalent timestamped pulse sequence). **SPI flash holds packed raw-GCR bits**, and the Macintosh ROM performs 6-and-2 encoding/decoding. The STM32's internal **sync generator** is a distinct, standard-track preparation function: it can provide canonical sync bits without re-encoding user data. Real-time RD pulse output must be independent of any format inspection or flash access.

### 10.6 Full sector structure — exact Macintosh Plus freshly formatted reference

**Purpose and evidence boundary.** This subsection is a **read-side reference fixture** for what the Macintosh Plus ROM's `Sony_CtlFormat` / `SONY_FORMATTRACK` / `SONY_DOFORMAT` sends through the IWM. It is useful for preparing, validating and replaying an initially formatted raw-GCR track. The **byte sequence transmitted during formatting** and the **final circular bitstream on a physical disk** are not identical concepts: the formatter deliberately transmits beyond one revolution and may adjust its gap length. Nor does an ordinary sector rewrite necessarily leave every pad/splice bit unchanged. Do not model the format operation as a request to the drive to write a sector record; these are consecutive GCR bytes that the IWM serializes onto WR.

**ROM evidence (inspect these routines, not merely a generic 800K diagram):**

- [`SONY_ADRMKTBL`, lines 2742–2752](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2742-L2752) — exact **27 transmitted bytes per sector's address/data prologue**; a following `$00` at template offset 27 is an **unused array pad**, *not* transmitted.
- [`Sony_CtlFormat`, lines 2755–2804](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2755-L2804) — formats 80 tracks, initially `GapSync = 7`, formats both sides unless one side is requested.
- [`SONY_FORMATTRACK`, lines 2883–2955](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2883-L2955) — checks the returned sector-0 address and adjusts `GapSync`, potentially rewriting the track.
- [`SONY_DOFORMAT`, lines 2958–3078](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2958-L3078) — **200 starting sync groups**, then gap, address/data template, **703 `$96` codewords** and **`DE AA FF FF`** for each sector.
- [`SONY_FILLINMARKS`, lines 3080–3131](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L3080-L3131) — header values, XOR checksum and sector ordering.
- [`SONY_WRDATAMKS` / `Sony_WrData`, lines 2531–2719](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2531-L2719) — ordinary **data-field-only** rewrite pattern; distinct from a format.

#### 10.6.1 Exact six-byte self-synchronization group

The ROM emits the following **48 serialized bits**, MSB first:

~~~text
SYNC6 (six ROM bytes) : FF 3F CF F3 FC FF
Serial bits           : 11111111 00111111 11001111 11110011 11111100 11111111
Equivalent bit pattern: (1111111100) × 4, then 11111111
~~~

Those **ten-bit** groups are deliberately misaligned relative to normal eight-bit GCR codewords; they let the IWM recover byte synchronization. `SYNC6` is a **bit pattern**, not six independently decoded `$FF` GCR bytes. It is constructed by the **Macintosh ROM**, clocked out by the **IWM**, and must be represented faithfully at bit level by the STM32. Address and data fields each have their own sync.

#### 10.6.2 Transmitted sequence when the Plus formats a track/side

Define `G = GapSync` (initially **7**; may change after the ROM checks the gap). The ROM enters the IWM write state using an initial loading byte, then runs this loop:

~~~text
ONCE AT THE BEGINNING OF THE FORMAT WRITE:
    200 × SYNC6                      ; startup/overrun sync; NOT a guaranteed
                                     ; permanent 200-group gap on the disk

FOR EVERY SECTOR IN THE ROM'S PHYSICAL INTERLEAVE ORDER:
    (G - 1) × SYNC6                  ; repeated inter-sector sync groups

    SYNC6                            ; final address sync group (template bytes 0–5)
    D5 AA 96                         ; address-field prologue (template 6–8)
    [T] [S] [H] [F] [X]              ; FIVE 6-bit values encoded as legal GCR bytes
    DE AA                            ; address end/slip marks (template 14–15)
    FF                               ; address OFF-PAD (template 16)

    SYNC6                            ; data-field sync (template 17–22)
    D5 AA AD                         ; data-field prologue (template 23–25)
    [S]                              ; sector ID, GCR-encoded (template 26)
    703 × 96                         ; freshly formatted zero tags/data/checksums
    DE AA                            ; data end/slip marks
    FF FF                            ; trailing WRITE-OFF padding

ONCE AFTER THE LAST SECTOR:
    Leave IWM write state; ROM reads the next address field to test fit.
~~~

**`[T] [S] [H] [F] [X]` are single eight-bit GCR codewords, not plain binary values:**

| Value | Meaning in the ROM format |
|---|---|
| `[T]` | Radial track, low six bits |
| `[S]` | Sector ID in the current interleave order |
| `[H]` | Side flag plus upper track bits |
| `[F]` | Format value (`$02` one-sided, `$22` two-sided), then encoded through `NiblTbl` |
| `[X]` | XOR of the four preceding *six-bit* values, then GCR-encoded |

The `703 × $96` means **699 GCR codewords for 12 zero tag bytes + 512 zero user bytes**, followed by **four GCR codewords for zeroed data checksums**. `$96` is the lookup-table encoding of six-bit zero. The separate, GCR-encoded **sector ID preceding those 703 codewords** is not generally `$96`. A subsequent ordinary write produces different 703 encoded bytes/checksums. The ROM transmits **27 template bytes** (offsets 0–26); the template's byte at offset 27 is **not output**.

#### 10.6.3 Exact gaps and their boundaries

| Location | Fresh format transmitted bits/bytes | Purpose |
|---|---|---|
| **Before an address mark** | `G × SYNC6` in total: `G-1` loop groups plus one from the template | Inter-sector gap plus address byte synchronization |
| **Address mark** | `D5 AA 96` | Next sector's address field |
| **Immediately after address `DE AA`** | Exactly **one `FF` pad byte**, followed by `SYNC6` | Address write-off pad and **independent** data synchronization |
| **Immediately before data `D5 AA AD`** | One `SYNC6` | Data-field byte synchronization |
| **Immediately after data `DE AA`** | Exactly **two `FF` pad bytes** | Allows final useful bytes to leave IWM before the write ends in ordinary writes |
| **Between data end and next address** | `FF FF`, then `G × SYNC6`, then next `D5 AA 96` | Normal formatted inter-sector boundary |

With the initial **`G = 7`**, there are **seven complete 48-bit sync groups** leading into each address mark (six emitted by the gap loop, one by the template). This is not seven `FF` bytes and it does not mean an unrecorded, pulse-free interval.

**One sector's transmitted format-loop length**, excluding the 200 initial groups and initial IWM setup byte, is:

~~~text
(G - 1) × 6  +  27  +  703  +  4  =  6G + 728 bytes
At G=7: 770 serialized bytes = 6,160 bit cells per sector entry
At 489,600 bit/s: approximately 12.582 ms per entry (before any physical wrap)
~~~

**Important circular-track distinction:** `200 × SYNC6` is a lead-in emitted at format time **before** the complete track's sectors. Because the medium rotates while the ROM keeps writing, later output can **overwrite part of this lead-in**. The ROM then searches for sector 0 and adjusts `GapSync` or rewrites to fit. It is therefore **incorrect to prepend 200 groups to a permanent cyclic track buffer** or to declare that the final gap from last sector to sector 0 must be exactly the original transmitted lead-in. The final seam/remaining gap must be determined from the chosen virtual revolution length and validated; a physical format's exact write-splice residue is not fully specified by the ROM byte tables.

#### 10.6.4 Physical sector order during formatting

`SONY_FILLINMARKS` constructs **2:1 interleave**: alternate entries in the first half of the IDs with entries in the second half. For `n` sectors, `k = ceil(n/2)`; the order is `0, k, 1, k+1, …`, stopping when all IDs have appeared.

| Zone (tracks) | Sectors per side | ROM format order |
|---|---:|---|
| 0–15 | 12 | `0 6 1 7 2 8 3 9 4 10 5 11` |
| 16–31 | 11 | `0 6 1 7 2 8 3 9 4 10 5` |
| 32–47 | 10 | `0 5 1 6 2 7 3 8 4 9` |
| 48–63 | 9 | `0 5 1 6 2 7 3 8 4` |
| 64–79 | 8 | `0 4 1 5 2 6 3 7` |

This is **physical stream order**, not the numeric order of logical 512-byte blocks presented to the operating system.

#### 10.6.5 STM32 internal sync generation — agreed standard-track policy

**[DECISION]** The SPI-flash cartridge stores **bit-packed raw 6-and-2 GCR track data**, not just decoded 512-byte sectors. The STM32 will have an **internal canonical `SYNC6` generator** for preparing and validating **standard, freshly formatted** track streams and for restoring known synchronization regions when operating in an explicitly standard/normalized track mode.

- Produce the **exact 48-bit `SYNC6` pattern**, with the appropriate group count and **without inserting/deleting bit cells during active RD playback**. This can be precomposed into a RAM track buffer or selected from a synchronized generator by track-region metadata; the implementation choice remains open.
- Keep the **five encoded header values, sector-ID byte, data payload and checksums** in their existing raw-GCR representation. Generating `SYNC6` must **not** require re-encoding the 6-and-2 data in the time-critical read path.
- Use a **validated track map** (track/side, bit positions, address/data sync regions, and stream length) when generating/replacing sync. Do **not** find and overwrite arbitrary `FF` runs: `FF` may be legitimate GCR data.
- Preserve the chosen **cyclic track duration**, all field prologues, and correct interleave. Do **not** append the 200 format-time lead-in groups to a finished track.
- **Never silently normalize unknown/nonstandard raw tracks.** Copy-protected layouts, deliberately unusual gaps, write splices and partially formatted media may depend on noncanonical content; replay raw bits unless a normalizing policy is explicitly selected. The shared policy is in **Part 3 / Section 11**; the exact rules for format capture and merging later writes belong to **Part 4 / Section 12**.

**[OPEN]** Define the final physical-to-virtual track seam, the representable gap-length ranges, whether to retain captured post-format write-splice bits, and how/when a standard-track map is marked valid. Verify the byte stream against Macintosh Plus formatting and subsequent readback with a logic analyser. **Readback acceptance** requires the IWM/ROM to recover `D5 AA 96`, all five valid header values, `DE AA`, then `D5 AA AD`, the sector ID, decoded zero tags/data and correct checksums at every zone.

### 10.7 The five zones and track wrapping

Apple 800K disks retain an approximately fixed bit rate by varying the **physical spindle RPM** between five track zones. We emulate the **resulting on-wire stream**, not the motor.

| Radial tracks | Sectors per side per track | Target source RPM | Approx. virtual revolution period |
|---|---:|---:|---:|
| 0–15 | 12 | 394 | 152.284 ms |
| 16–31 | 11 | 429 | 139.860 ms |
| 32–47 | 10 | 472 | 127.119 ms |
| 48–63 | 9 | 525 | 114.286 ms |
| 64–79 | 8 | 590 | 101.695 ms |

There are **80 tracks per side** and **800 sectors per side**, therefore **1600 × 512-byte user sectors = 819,200 bytes (800 KiB)** across two sides. **400K media** uses one side of the same zone/track geometry (800 sectors); presenting a 400K formatted disk is distinct from emulating a 400K-only mechanism.

**[ROM]** The exact **2:1 sector interleave** for freshly formatted tracks is listed in `10.6.4; its sequence varies by sector count/zone. The ROM **scans a continuous track stream**, not randomly addressed sector records. The format-time 200-group lead-in is **not** a permanent virtual track prefix (`10.6.3).

**[DECISION]** Use a logically **cyclic, continuous track stream** for each selected radial track and side. After the final encoded bit cell, wrap to the first bit cell without producing a false sync/mark or illegal long zero gap. When the selected track or side changes, switch the backing stream coherently and satisfy /READY before advertising a new valid stream. One can choose an arbitrary *initial virtual phase* (no physical rotational-latency simulation required), but once running the stream must behave like a valid ongoing sequence, including gaps and wraparound.

**Cross-check with TACH (Part 1 §8; Part 3 §11):** The Plus ROM **does read TACH even for 800K drives**. For a standard track, the STM32 derives the target TACH frequency from the **logical track descriptor's sector count** (12→394 Hz, 11→429 Hz, 10→472 Hz, 9→525 Hz, 8→590 Hz); this is an electrical compatibility lookup, **not a physical motor simulation or a second GCR calibration**. The entire raw-GCR stream continues at the **single global bit-cell rate**. Ensure the resulting TACH rate and cyclic track duration are acceptably coherent.

**Engineering estimate, not a buffer commitment:** A rate of about **489.6 kbit/s** corresponds to **61.2 kB/s**. A revolution thus holds roughly **9.3 kB** worth of bit cells on the outermost zone or **6.2 kB** on the innermost; these totals include sync/gaps and are not synonymous with user-sector storage size. One complete currently selected *encoded* track could therefore plausibly fit in approximately **10 KiB** of bit-packed storage, plus buffering/metadata. Actual worst-case buffering, head changes and phase scheduling remain for measurement.

### 10.8 Macintosh Plus ROM routines and error cases

The Plus's ROM does **not** send “read sector N” to the Sony drive. Instead, it commands a track and head, then **looks for the next matching address field within the serial stream**. Its low-level algorithm is visible in the disassembly:

| ROM routine | Source location | Purpose |
|---|---|---|
| `Sony_RdAddrSetup` | [2094–2106](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2094-L2106) | Select RDDATA0 or RDDATA1 from the current side |
| `Sony_RdAddr` | [2113–2223](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2113-L2223) | Poll IWM `q6L`, find `D5 AA 96`, decode track/sector/side/format and validate address checksum + slip |
| `Sony_RdData` | [2234–2506](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2234-L2506) | Find `D5 AA AD`, decode 12 tags + 512 bytes, validate checksums + slip |
| `SONY_DENIBLTBL` | [2508–2530](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2508-L2530) | 8-bit GCR codeword to 6-bit value |
| `SONY_WRDATAMKS` | [2531–2539](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L2531-L2539) | Reference for correct self-sync and data-mark construction |
| `Sony_ReSeek` | [832–897](https://github.com/jonathanschilling/mac_rom/blob/main/plus/resources/res_drvr_sony.s#L832-L897) | Seek desired track, validate speed/zone, then read address fields and sectors |

**[ROM]** `Sony_RdAddr` first looks for **three complete nibbles** and then searches for the **`D5 AA 96`** address mark, with a bounded mark search (new-interface drive code uses a shortened count). It decodes the header's track, sector, side, format and XOR checksum, then requires the two `DE AA` slip marks. `Sony_RdData` subsequently searches for `D5 AA AD`, reads the data and validates its three running checksums.

Relevant ROM-side errors include:

| ROM error | Value | What a nonconforming stream can cause |
|---|---:|---|
| `noNybErr` | **−66** | No usable GCR nibbles detected |
| `noAdrMkErr` | **−67** | No valid address field mark found |
| `badCksmErr` | **−69** | Address checksum mismatch |
| `badBtSlpErr` | **−70** | Address end/slip marks invalid |
| `noDtaMkErr` | **−71** | Data field mark missing |
| `badDCksum` | **−72** | Data checksum incorrect or invalid GCR nibble |
| `badDBtSlp` | **−73** | Data end/slip marks invalid |

**[DECISION]** Our normal bootable/format-correct media path must generate a complete, self-consistent stream, not merely avoid `noNybErr`. Blank-media and image-corruption behaviour should be defined consciously, with the ROM's retry/format path evaluated, rather than masking all errors as success.

### 10.9 Streaming model, transitions and storage interaction

The **read-data engine** should be conceptually separated into *raw-track retrieval*, *optional standard-track sync construction/validation*, *bitstream scheduling* and the *physical RD output path*:

~~~text
Removable SPI-flash cartridge (packed raw-GCR bits + track metadata)
                              │
                 Load selected track/side into RAM
                              │
          Inspect/validate address and data boundaries
            [standard-track mode, when applicable]
                              │
        Internal canonical SYNC6 generator/normalizer
       (only known sync regions; never re-encode payload)
                              │
              Prepared cyclic bit-packed track
               (known bit length and virtual phase)
                              │
              Deterministic pulse-event scheduler
                 (1 bits = pulses; 0 = no pulse)
                              │
           Multiplexed, tri-state-capable RD output
                       Macintosh IWM
~~~

Unknown/nonstandard raw tracks bypass canonical sync replacement. An ordinary read is **not** a request to generate a new sector from decoded data; it replays the prepared raw stream.

**State and event expectations:**

- **At motor-on:** Start the virtual stream/tach clock as appropriate. Once current track data is prepared, allow `/READY=0`. Do not add artificial spin-up time.
- **At step or side change:** Prevent data from being falsely advertised as ready while a prior side/track's stream is still installed. Install the new stream and corresponding zone timing, then assert ready.
- **At status selection:** RD must present the selected status; *the stream clock need not be restarted merely because the host momentarily reads status*. Preserve sensible phase continuity where feasible.
- **At head-data selection:** Deliver the current selected stream, not a fixed level, and do not treat SEL itself as a command strobe.
- **At write gate:** A true drive suppresses/replaces read data during writing and resumes subject to source timing. Exact gate-to-read resumption is part of **future Part 4 / Section 12**; the read generator must expose suitable pause/resume/handoff hooks.
- **At eject / cartridge removal / fault:** Stop exposing old disk data; mark logically not ready and absent as appropriate (Part 1 `9).
- **At end of encoded track:** Repeat the valid bitstream seamlessly. Avoid injecting fake index pulses: the Sony common interface has no separate index pin and the GCR read path uses field marks to find sectors.

**[OPEN]** The best minimal-latency design may preload both sides of the currently selected radial track, hold a whole bit-packed track, or use an event list of pulse deltas. Select after measuring STM32 RAM, CPU/DMA bandwidth, and SPI flash worst-case stall duration. Keep the data presented on RD independent of flash stalls after it is declared ready.

**[OPEN]** Nonstandard disk images with intentionally malformed fields, weak bits or copy protection cannot be represented faithfully by a simple 512-byte sector-per-track image. Decide separately whether the initial project targets **standard 400K/800K images only** or must preserve arbitrary track-level bitstreams. This is a scope decision, not a hidden assumption.

### 10.10 Potential STM32 timing architecture

**No MCU/peripheral choice is approved yet.** A plausible division of responsibility is:

| Component | Time critical? | Possible implementation |
|---|---|---|
| Media/image retrieval and track cache | No, if done before ready | SPI peripheral + DMA/foreground |
| Standard-track sync-generation, region-map inspection and validation | No, if prepared before ready | Canonical 48-bit sync template + raw-GCR metadata; no routine 6-and-2 payload re-encoding |
| Prepared cyclic stream phase and zone timing | Yes, for smooth output | Hardware timer plus coherent buffer state |
| **Pulse rise and fall (0.3–0.8 µs wide)** | **Yes** | Timer output compare / DMA event stream / external gating |
| Data-source and status multiplexer | **Yes (≤0.5 µs source timing)** | Verified fast hardware mux / GPIO+buffer, subject to bench measurement |
| RD output-disable on `/ENBL` high | **Yes (≤0.5 µs)** | Prefer asynchronous hardware output-enable gating if needed |
| Tachometer signal (Part 1 `8) | Yes | Independent hardware timer, common zone model |
| Write-data capture (future Part 4 / §12) | Yes | To be investigated separately; cannot assume read timer is shareable |

A CPU-driven interrupt for *every GCR transition* may approach hundreds of thousands of events per second at worst, leaving little deterministic time for other work. **This is a workload warning, not a proven STM32 failure.** We should evaluate DMA-assisted timer updates or a small external timing/mux stage before approving a part.

**Preliminary pulse scheduler pseudocode (not executable firmware):**

~~~text
prepare_track(track, side, media):
    raw_bits, metadata = read_packed_GCR_track_from_flash(track, side)
    if metadata.track_mode == STANDARD and validate_region_map(raw_bits, metadata):
        raw_bits = normalize_only_mapped_sync_regions(raw_bits, SYNC6)
        assert bit_length_is_unchanged(raw_bits)
    else:
        preserve_raw_bits_exactly()
    atomically_publish_stream_and_phase_if_selection_still_matches()

hardware_timed_output:
    for each bit cell in repeating encoded_track_bitstream:
        if bit == 1:
            emit_RD_pulse_with_verified_width()
        else:
            emit_no_pulse()
    on track end:
        wrap seamlessly to bit 0

on CA/SEL source change:
    select static sense / TACH / correct head stream; avoid drive-to-host glitches

on /ENBL inactive:
    force RD high impedance by hardware-gated enable

on media/track not ready:
    do not advertise /READY low; prevent stale stream being presented as valid
~~~

The precise bit-cell clock, pulse polarity, high/low levels under inactive read selection, edge alignment, track wrap phase, and gating against host write activity require **bench verification**. Do not assume the STM32's fastest interrupt latency is equivalent to its GPIO output toggle rate.

### 10.11 Verification and performance acceptance

| Verification | Initial pass criterion | Evidence |
|---|---|---|
| Project `0x8` or `0x9` selected, drive enabled | Correct head's data pulse stream, not status bits | [SONY]/[ROM] |
| Drive disabled | RD high impedance within **0.5 µs** | [SONY] |
| RD source/selector change | Correct new source promptly; validate **0.5 µs** bound | [SONY] |
| Individual data pulses | **0.3–0.8 µs width** at connector | [SONY] |
| Legal GCR pulse spacing | Nominal **2, 4, 6 µs** pattern and transition tolerance | [SONY]/[IWM] |
| Nominal bit rate | **489,600 bit/s / 2.042484 µs** project baseline; verify with logic analyser | [DECISION; bench pending] |
| 800K track zones | 12/11/10/9/8 sectors, 80 tracks/side, correct mark/checksum layout | [SONY] |
| IWM reads `q6L` | Byte-complete high bit and expected GCR code bytes produced | [IWM]/[ROM] |
| Self-sync and cyclic wrap | ROM locks to `D5 AA 96` at arbitrary phase; validate **no permanent 200-sync lead-in** and no invalid wrap seam | [ROM]/[DECISION] |
| Address data correctness | Correct track/side/sector/format and XOR checksum | [ROM] |
| Data field correctness | Fresh format reads zero tags + 512 zero bytes from 703 × `$96` after encoded sector ID; normal writes read valid 6-and-2 data/checksums | [ROM] |
| Both heads | Change side without incrementing radial track; correct data selection | [ROM] |
| Status-to-data switch | No malformed leading pulses or unwanted command execution | [DECISION] |
| Slow/busy flash operation | RD timing stays within bounds while ready | [DECISION] |
| Blank/unformatted media | Defined, reproducible behaviour; ROM doesn't mount nonexistent data | [OPEN] |
| Timing under full load | Worst-case jitter and pulse width measured with SPI flash, TACH and status tasks active | [OPEN] |
| Boot validation | Macintosh Plus ROM boots from a standard 800K image and reads both sides correctly | [ACCEPTANCE] |

**Recommended test order:** 1) hardware pulse-width/gap fixture; 2) synthetic repeating `FF` and short known GCR mark streams; 3) full one-side track at all five zone rates; 4) IWM+ROM address/data parse; 5) full 800K image read and boot; 6) heavy SPI activity, repeated selector/track/head changes; 7) 400K media path.

### 10.12 Open questions and design decisions

**Established now:**

- **[DECISION]** Section 10 is the only numbered protocol operation in **06 — Part 2**. See **[Part 3 / Section 11](06-part-3-media-architecture.md)** for the common cartridge/timing/metadata architecture and future Part 4 / Section 12 for serial write capture.
- **[DECISION]** The STM32 generates genuine GCR-like **RD electrical pulses** through the unchanged Macintosh IWM.
- **[DECISION]** One logical-track metadata model (sector count, valid bit length) and **single global GCR calibration** must agree with Part 1's TACH and /READY. The STM32 does not model a physical motor or maintain five separate GCR clocks.
- **[DECISION]** Cartridge storage is **bit-packed raw GCR**; the STM32 may generate **canonical 48-bit SYNC6** internally in mapped standard-track sync regions, without re-encoding sector payloads or silently modifying nonstandard tracks.
- **[ROM]** Fresh format detail is specified in `10.6: 200 transient lead-in groups, per-sector GapSync groups, exact address/data pad bytes, 703 `$96` formatted-data codewords and 2:1 interleave.
- **[DECISION]** No mechanical or acoustic simulation and no deliberately added rotational/seek delays solely for realism.
- **[DECISION]** No connection to host pins 9 or 20.
- **[DECISION]** Leave write capture/erase/write-gate timing for **06 — Part 4, Section 12**.

**Research/implementation gates before hardware selection:**

1. **Bench-validate the chosen 489,600 bit/s / 2.042484 µs nominal cell** on a real Macintosh/IWM with a logic analyser (and verify read and write pulse tolerances); the choice is settled, measurement is pending.
2. Confirm exact **RDDATA idle polarity, pulse shape and line loading**, and prove ≤0.5-µs mux/output-disable behaviour with the selected STM32/buffer architecture.
3. Cross-check **`3.4.1.2 T1/T2/T3/T6`** edge relationships on original hardware, particularly head switching and write-gate recovery.
4. Define how to implement **continuous cyclic streams** while not imposing unnecessary initial rotational latency, and determine head-to-head phase relationship if relevant.
5. Define packed **raw-GCR flash layout**, bit-length metadata, sync-region map and the explicit boundary between **canonical standard tracks** and **untouched nonstandard raw tracks**; avoid silent changes to copy-protected data.
6. Validate **fresh-format stream and raw playback** against known ROM fixtures (including 200 transient lead-in groups, `GapSync` adjustment, both prologues, pads, checksums, slip marks and byte-boundary recovery); no routine payload re-encoding in STM32 is assumed.
7. Measure **worst-case timer/DMA jitter, CPU loading, memory footprint and SPI flash stalls** for candidate STM32 devices.
8. Determine exact behaviours for **unformatted media, write splice, speed-zone transition during active output and drive re-enable**.
9. Define the final boundary between **Part 2 read engine**, **Part 3 shared media state**, and **Part 4 write capture**, including ownership of RD, track buffers and persistence.

**Document history**

- **2026-11-09:** Linked new **Part 3 / §11** shared cartridge/timing/metadata contract; moved the planned detailed writing section to **Part 4 / §12** and roadmap to §§13–18. Clarified logical sector-count-to-TACH lookup versus the common GCR clock.
- **2026-11-09:** Expanded `10.6` with the exact **freshly formatted GCR write stream** as a read-side reference; fixed address/data pad/gap details, `GapSync`, transient 200-group lead-in and 2:1 interleave. Settled STM32 **canonical sync generation for mapped standard tracks**, preserving raw-GCR flash storage and nonstandard-track replay. Updated chosen **489,600 bit/s** timing baseline; bench measurement remains pending.
- **2026-11-09:** Created **06 — Part 2**, giving Section 10 its own research and implementation specification. Original **06** split into **06 — Part 1** (Sections 1–9); future serial-write section was originally assigned to Part 3 (subsequently moved to Part 4 / §12 after the shared architecture part was inserted). Timing and storage design deliberately remain under verification.
