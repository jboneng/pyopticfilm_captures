# OpticFilm 8200i SE — USB protocol & behaviour (capture bible)

**Status:** synthesis of the SilverFast / vendor-driver capture campaign  
**Device:** Plustek OpticFilm 8200i SE · USB `07b3:1825` · Genesys **GL128** (`bcdDevice 0x0702`)  
**Host app for all behavioural sessions:** SilverFast 9 + Plustek `usbscan` (not WinUSB)  
**Dates:** 2026-07-29 (sessions 01–06) · 2026-08-03 (sessions 08–12)  
**Audience:** anyone aligning `plusteklib` (`Gl128`, `model_8200i_se`, `session_gl128`) with real hardware

This document consolidates what the captures proved. Per-session detail lives in
each folder’s `NOTES.md`. Motor recipes are also summarised in [`MOTOR.md`](MOTOR.md).
Running log: [`SESSION_LOG.md`](SESSION_LOG.md).

---

## 1. One-paragraph verdict

The SE speaks the same Genesys **vendor-request framing** as the GL845 path in
plusteklib, but its **register map and scan choreography are GL124-family**, not
GL845/8200i-SANE. A normal colour (or IR) job is a fixed-shape pipeline:
cold/warm init → AFE + shading (always 16-bit) → two fast feeds from home →
image acquire with `AGOHOME` (8-bit USB) → carriage parks as a side-effect.
Infrared (iSRD) is that whole pipeline run a second time with the white lamp off
and IR LED on. There is no standalone reverse-home in any capture.

plusteklib’s early SE motor path **did not** match this and caused grinding;
motor moves remain gated until a capture-faithful re-test.

---

## 2. Device & capture method

| Field | Value |
|-------|--------|
| USB ID | `07b3:1825` |
| Product string | `Film Scanner (A2F)` (docs sometimes said A2M) |
| `bcdDevice` | `0x0702` → GL128 |
| Driver for captures | Vendor WIA `usbscan` |
| Capture stack | Windows 11 + Wireshark/USBPcap; scanner on `\\.\USBPcap1` |
| Decode tooling | `scripts/_se_capture_decode.py`, `scripts/extract_se_captures.py` |

**Required USBPcap option:** “Inject already connected devices descriptors into
capture data” — attribution depends on seeing the device descriptor.

Do **not** run plusteklib motor/scan (or GL845 sequences) while capturing vendor
traffic.

---

## 3. Campaign map (first → last)

| # | Folder | Result | What it established |
|---|--------|--------|---------------------|
| 01 | `01_enumerate/` | **ok** | Descriptors: bulk IN `0x81` / OUT `0x02` @ 512, int IN `0x83` |
| 02 | `02_cold_boot_open/` | **ok** | Framing compatible; init blast; FE path `0x51`/`0x5d`/`0x5e`; no soft reset |
| 03 | `03_home_or_preview/` | **ok** | Full pipeline at 1200 dpi preview; status `0x101`; lamp; motor slopes; first geometry point |
| 04 | `04_color_1800/` | **ok** | 1800 dpi colour; geometry formulas exact; 8-bit image / 16-bit shading |
| 05 | `05_ir_1800/` | **ok** | iSRD = second full pipeline; IR = `0x37` bit 2 + lamp off |
| 06 | `06_color_high_dpi/` | **ok** | 3600 dpi; third geometry confirmation; memory/slopes constant |
| 13 | `13_ppi_ladder/` | **ok** | Full SilverFast PPI set 150–7200; DPISET floor; no STAGGER at 7200 |
| 07 | `07_calib/` | **pending** | Optional; calib already present inside 03–05 |
| 08 | `08_midtravel_home/` | **ok** | Cancel during **image**; park via armed `AGOHOME` |
| 09 | `09_y_crop_pair/` | **ok** | Feed1 always 28292; feed2 tracks Y (13128 top / 20232 bottom) |
| 10 | `10_back_to_back/` | **ok** | Warm RGB→IR / next scan: always full feeds from home after park |
| 11 | `11_bit_depth_1800/` | **ok** | Depth regs `0x33`/`0xAF`; SilverFast “48-bit HDR” still 8-bit USB |
| 12 | `12_abort_during_feed/` | **blocked** | SilverFast never aborts mid-feed; evidence pcap kept |

---

## 4. USB framing (Genesys class)

Compatible with existing `GenesysUsbProtocol` framing:

| Op | Setup (summary) |
|----|-----------------|
| Register write | `0x40` / `0x04` / `wValue 0x0083` (+`0x100` high addr), payload `(addr, value)` pairs |
| Register read | `0xc0` / `0x04` / `wValue 0x008e`, `wIndex = 0x22 + (addr<<8)`, 2 bytes; link byte **`0x55`** |
| High address | Fold `wValue` bit 8 into address (e.g. `0x101`) |
| AHB bulk preamble | `wValue 0x0082`, 8-byte LE address + size |
| `write_0x8c` | `bRequest 0x0c`, `wValue 0x008c` |

**Vendor vs plusteklib differences (functional, not framing):**

- Writes are **batched** (up to ~32 pairs per transfer); plusteklib often sends one pair per URB.
- Bulk image stream uses different `wIndex` values than plusteklib historically used for RAM write (`0x00` / `0x08` seen for reads; see session notes).
- Extra 1-byte `REQUEST_REGISTER` probes selected by `wIndex` (`0x00`, `0x18`, `0x20`, **`0x21`** for feed done).

---

## 5. Chip family & register map

The SE is **not** a GL845 clone for scan control:

| Area | SE (observed) | Do not reuse from GL845 blindly |
|------|---------------|----------------------------------|
| Frontend | `0x51` index → `0x5d` high / `0x5e` low (16-bit AFE) | `0x3a`/`0x3b` FE writes |
| Memory layout | `0xe0`–`0xf8` block | Different GL845 layout |
| Status | **`0x101`** | `0x41` is never used as status |
| GPO | `0xa6`–`0xa9` | GL845 `0x6b`–`0x6f` unused here |
| Soft reset | **Never** (`0x0e`–`0x10` never written on cold open) | Assumed reset sequences |

### Important control registers

| Addr | Role | Notes |
|------|------|--------|
| `0x01` | Scan control | `SCAN`, `SHDAREA`, `DVDSET`, … Image arm often `0x23`; cancel clears to `0x22` |
| `0x02` | Motor | `AGOHOME=0x20`, `MTRPWR=0x10`, `FASTFED=0x08` |
| `0x03` | Lamp / XPA | Bit 4 = white lamp (`0x30` on, `0x20` off). **`XPASEL` never set** in any session |
| `0x0d` | Clear counters | `0x07` used in **scan start** recipe; **not** before fast-feed `START` |
| `0x0f` | START | `0x01` launches configured op |
| `0x25`–`0x27` | `LINCNT` | 24-bit BE |
| `0x2c`–`0x2d` | `DPISET` | 16-bit BE |
| `0x33` | Depth A | `0x04` = 16-bit, `0x1F` = 8-bit (image) |
| `0x37` | IR / mixed | Bit 2 = IR LED (RMW). Upper bits often HW status on read |
| `0x3d`–`0x3f` | `FEEDL` | 24-bit BE feed distance |
| `0x82`–`0x87` | `STRPIXEL` / `ENDPIXEL` | 24-bit BE |
| `0xAF` | Depth B | `0x46` = 16-bit, `0xFF` = 8-bit (image) |
| `0x101` | Status | Polled heavily; home-related bit observed in warm gaps |

---

## 6. Cold open / init (session 02)

After power-cycle + SilverFast open:

1. Framing probes and link checks.
2. Large **batched init** (~116 registers) — values live in `model_8200i_se` / decode JSON.
3. Small AHB blobs (including a 34-byte write with a lone `0x33` at offset 32 — unexplained, identical in colour and IR).
4. Frontend programming via GL124 path.
5. Button LED on; **no** lamp strike; **no** motor in this session.

No soft-reset register sequence.

---

## 7. Status & completion signalling

- Primary status: read **`0x101`**.
- Observed values (context-dependent; not a full bitfield map):
  - Busy / moving often around `0xa5`, `0xd5`, …
  - Idle / park progression e.g. `0xa5` → `0xad` → `0xec` after `AGOHOME`
  - Home bit seen set when parked between back-to-back passes (session 10)
- Fast **feed completion** is also signalled by vendor probe **`wIndex=0x21` → `0x04`**
  (fallback: watch `0x101`).
- Other probes: `wIndex 0x20` often returns `0x55` (link-like); `0x18` after AHB.

---

## 8. Lamp and infrared

### White lamp

- `0x03` bit 4: on during visible acquisition (`0x30` with other held bits).
- Cleared for IR (`0x03 = 0x20`).

### iSRD / IR (session 05)

iSRD is **not** a half-scan or GPO mux. It is:

1. Full colour pipeline (including AFE + shading), then  
2. Full pipeline again with:
   - white lamp off (`0x03` bit 4 clear)
   - IR LED on via **`0x37` bit 2** (read-modify-write; e.g. toward `0xb4` / `0xf4`)

Geometry and slope tables are identical between passes → pixel-aligned RGB + IR.
GPO block does **not** change for IR (unlike SANE GL845 8200i).

Session 10 (HDRi): each UI scan produces **two** feed+image cycles (RGB then IR),
each parking via `AGOHOME` before the next.

---

## 9. Geometry (DPI → registers)

Native optical rate: **7200 dpi**. Decimation factor:

\[
f = 7200 / \mathrm{dpi}
\]

Verified **exact** at every SilverFast PPI ≥ 600 (session 13); below 600,
`DPISET` **floors at 100** (ASIC programmed as 600):

| Quantity | Formula |
|----------|---------|
| `DPISET` | `max(dpi, 600) / 6` |
| `LINCNT` | `output_lines × f` with `f = 7200 / max(dpi, 600)` |
| `ENDPIXEL − STRPIXEL` | `output_width × (7200 / max(dpi, 600))` |
| Host averaging | ASIC returns `LINCNT` native-rate lines; host averages `f` lines → ASIC-rate line; PPI < 600 then block-downsamples |
| Horizontal | ASIC already decimates width via `DPISET` |

**Calibration / shading** always run at full native segment rate (`DPISET = 1200`
in the sense of native calib), **16-bit** (6 bytes/px).  
**Image** at requested DPI (ASIC ≥ 600), **8-bit** USB for SilverFast colour modes (3 bytes/px).

Only a small set of registers change with DPI; memory layout `0xd0`–`0xf8` and
both motor slope tables are **resolution-invariant** across tested DPIs.

Per-channel exposure table uploads at `0x10000000` / `0x4000` / `0x8000` are
uniform `14000 / f` across 256 entries (`f = 7200 / max(dpi, 600)`).

`STAGGER` (`0x01` bit 4) was **never** set at any captured PPI including **7200**.

---

## 10. Bit depth (sessions 04 + 11)

| Mode | `0x33` | `0xAF` | Bytes/px (RGB) |
|------|--------|--------|----------------|
| Shading / calib | `0x04` | `0x46` | 6 (16-bit) |
| Colour image | `0x1F` | `0xFF` | 3 (8-bit) |

Session 11 compared SilverFast **48 → 24 Bit** vs **48 Bit HDR RAW** at 1800:

- Image-pass register window: **byte-identical**
- Bulk totals: **identical**
- Both image at `0x1F` / `0xFF`

So “48 Bit HDR” is a **host file** depth, not a 16-bit USB image stream.
No capture yet shows a colour **image** pass at 16-bit on this unit under SilverFast.

---

## 11. Motor behaviour (critical)

### Positioning model (from home)

```text
feed(28292)              # constant reference — every session
feed(y_start_steps)      # crop / Y dependent
image: FEEDL=1, 0x02=0x30  # AGOHOME|MTRPWR — parks when done
```

| Feed 2 examples | Steps |
|-----------------|-------|
| Preview / top crop (03, 08, 09a, 10) | **13128** |
| Full-ish colour (04) | **13704** |
| Bottom crop (09b) | **20232** |

Delta top→bottom (same height crop): **7104** steps. Exact mm↔steps formula is
not closed yet; do **not** use GL845 `geometry.starty` from `y_offset_ta_mm`.

### There is no standalone home

Return-to-home is **only** observed as `AGOHOME` on the image (or cancel-during-image)
pass. Invented `FEEDL=0` + reverse seek caused hardware grinding.

### Fast-feed recipe (ordered)

See [`MOTOR.md`](MOTOR.md). Essentials:

1. Setup block (scan clear, FEEDL, exposure, window, `0x02=0x18`, …)
2. Upload **fast** slope table (512 B) to `0x1000c000` and `0x10010000`
3. `0x0f = 0x01` **only** — do **not** write `0x0d` here
4. Wait probe `wIndex=0x21` → `0x04` (or status)
5. `0x02 = 0x08`, then `FEEDL = 1`

Slow slope table is for shading passes, not these repositions.

### Warm / back-to-back (session 10)

After park at home, the next pass (IR or next scan) **still** does `28292` +
`y_start` — no mid-frame short reposition.

### Cancel during image (session 08)

Exact recipe (also session 03 natural return-home; `08b`/`08c`):

```text
0x03 = 0x30, 0x20
0x01 = 0x22
0x03 = 0x10, 0x00, 0x20, 0x30, 0x20, 0x30
```

Leave `0x02` / `FEEDL` alone — `AGOHOME` already armed parks the carriage.
`Gl128.stop_motor` replays this when `AGOHOME` is set.
### Mid-feed abort (session 12)

**Unavailable in SilverFast 9.** Even “cancel as fast as possible” finishes both
feeds; full-scan Cancel stays disabled. No capture-derived `stop_motor` recipe.

---

## 12. End-to-end scan pipeline shape

Mermaid flowcharts (one-pass, iSRD two-pass, colour vs IR): [`FLOW.md`](FLOW.md).

Fixed shape across DPI (values change, sequence does not):

1. **Init / re-init** (cold blast or warm `0x02=0x78`-style blocks)
2. **AFE calibration** — runtime search of offsets (`FE 0x02`–`0x04`) and gains (`0x05`–`0x07`)
3. **Shading** — 16-bit, slow slope, full-width native-rate windows
4. **Position** — feed 28292, then feed `y_start`
5. **Image** — geometry + depth 8-bit + `FEEDL=1` + `0x02=0x30` + scan-start recipe:
   memory/geometry/AFE already set → `0x0d=0x07` → `0x01 |= SCAN` → `0x0f=0x01` →
   poll `0x101` / bulk IN
6. **Park** via `AGOHOME` as bulk completes (or on cancel after SCAN clear)
7. Optional **IR pass**: repeat from calib/shading with lamp/IR deltas

---

## 13. Incident: grinding (why motor is gated)

An early plusteklib SE path combined:

- GL845-style `starty` (~8078 from `y_offset_ta_mm`) as a pre-scan feed  
- A short feed recipe (`0x0d` before feed START)  
- An invented standalone home (`FEEDL=0` + `AGOHOME|MTRPWR|FASTFED`)

None of that appears in vendor traces. Hardware survived after power unplug +
SilverFast recovery; **`scan_ready=False`** and **`_motor_moves_enabled=False`**
remain until a careful ladder re-test.

---

## 14. What plusteklib should mirror (alignment checklist)

| Topic | Capture truth | Library target |
|-------|---------------|----------------|
| Framing | Genesys vendor requests | Keep; batch writes optional |
| FE path | `0x5d`/`0x5e` | SE-specific (done) |
| Status | `0x101` | Not `0x41` (done) |
| Position | 28292 + `y_start` | Not `geometry.starty` (done; crop via `feed_to_scan_steps_for_area`) |
| Home | Side-effect of `AGOHOME` | Refuse invented reverse-home (done) |
| Image depth | `0x1F` / `0xFF` | `DEPTH8` on image; upsample host-side (done) |
| Shading depth | `0x04` / `0x46` | `DEPTH16` for calib (done) |
| IR | `0x37` bit 2 + lamp off | Full second pipeline (done) |
| Abort image | Lamp strobe + `0x01=0x22`; leave `AGOHOME` | `stop_motor` when AGOHOME armed (done) |
| Motor gate | HW re-test pending | `scan_ready=False`, `_motor_moves_enabled=False` |

---

## 15. What we still do not know

### Behaviour / protocol gaps

1. **Absolute origin of the feed scale**  
   mm↔steps and the LINCNT coupling are now proven: feeds are 14400 steps/inch,
   `travel_mm = LINCNT × 25.4 / (4 × asic_dpi)`, and every capture stops at or
   before the window end 27636 (sessions 03 and 09b land on it). The absolute
   scale comes from the film, not the protocol: the session-13 crop is 36.06 ×
   24.24 mm, a 3:2 35 mm frame. What is still unmeasured is where step 0 sits
   relative to the film holder — the window is modelled as 25.59 mm starting at
   `feed2=13128` ([MOTOR.md](MOTOR.md)).

2. **Mid-feed motor stop**  
   No vendor recipe; SilverFast cannot demonstrate it. Other apps untested.

3. **True 16-bit USB colour image**  
   Never observed under SilverFast 24-bit / 48-bit HDR modes. Unknown if any host
   path programs image `0x33=0x04` / `0xAF=0x46`.

4. **Full `0x101` bitfield**  
   Enough values for home/busy/park heuristics; not a complete documented map.
   Park wait uses `is_at_home` + motor idle.

5. **Semantics of status probes `wIndex 0x18` / `0x20`**  
   Observed patterns only; `0x21`→`0x04` is the important feed-done signal.

6. **34-byte AHB blob (`0x000fff00` / `0x000fff01`)**  
   Byte 32 = `0x33` always; not IR-related; purpose unknown.

7. **Several DPI-tied registers** (`0x029`/`0x02a`, `0x02e`, `0x120`, …)  
   Change with DPI but not fully explained.

8. **7200 dpi / `STAGGER`**
   Captured in session 13 — `STAGGER` remains clear; `DPISET=1200`,
   `LPERIOD=15963`, `0x2B=0x17`.

9. **Session 07 dedicated calib**  
   Not captured; likely redundant with embedded calib in 03–05.

10. **Abort with `AGOHOME` clear mid-frame**  
    No capture of a reverse seek from mid-travel without prior `AGOHOME`.

11. **Other host software**  
    QuickScan / VueScan IR and cancel behaviour unknown.

12. **Runtime AFE offset/gain search + ASIC shading**  
    **Steps 1–3 done:** dichotomy AFE (`Gl128.search_afe`), stationary shading
    measure/upload (`run_asic_shading` → `0x10014000`), host stretch skipped when
    cache entry has `asic_shading`. See [`CALIB.md`](CALIB.md). Motor remains
    gated; shading widths from session 13 (+ 03/04/06 for 1200/1800/3600).

13. **Bulk `wIndex` policy edge cases**  
    Double-check production bulk reads against decode if issues appear.

### Closed in library (vs earlier gaps)

- Image vs shading depth (`DEPTH8` / `DEPTH16`)
- Crop-dependent second feed
- Scan-start order `0x0d` → SCAN → `0x0f`
- Image cancel/end lamp strobe on `0x03` + `0x01=0x22` (sessions 03/08)
- AGOHOME park wait after image / cancel
- Calib `DPISET=1200`, clear `DVDSET`, no reverse-home
- Slow vs fast slope tables
- Warm prepare `0x02=0x78` before positioning
- Stationary AFE dichotomy (`search_afe` / `acquire_afe_strip`)
- ASIC shading upload (`run_asic_shading` → `0x10014000`)

### Library / HW validation gaps

- Capture-faithful motor code exists behind the gate; **hardware ladder not re-run**
  since the grinding incident.
- Offline “replay assert vs pcap window” tests are desirable but not complete
  for every phase.

---

## 16. Safe re-test ladder (when aligning on hardware)

Only with operator consent, hand on power:

1. Open / status / lamp on (no motor)  
2. Arm motor gate  
3. Single capture-faithful `feed(small)` only if ever proven — prefer known
   `28292` only when ready for full motion  
4. Full `28292` → `y_start` → tiny or preview image with `AGOHOME`  
5. Compare register order to unit tests / pcap snippets  

If anything sounds wrong: power off, do not invent a reverse-home.

---

## 17. Source index

| Document | Contents |
|----------|----------|
| This file | Campaign synthesis |
| [`MOTOR.md`](MOTOR.md) | Feed/home recipes |
| [`SESSION_LOG.md`](SESSION_LOG.md) | Host setup, tooling fixes, numbered findings |
| `0N_*/NOTES.md` | Session-level register detail |
| `decoded/` | Extracted JSON / tables |
| `docs/gl128-bringup.md` | Capture *method* (may lag gated motor status) |
| `plusteklib/asic/gl128.py` | Intended SE ASIC driver |
| `plusteklib/device/model_8200i_se.py` | Model constants / tables |
| `plusteklib/scan/session_gl128.py` | Scan session configure |

---

*Last updated: 2026-08-03 — after sessions 01–06 and 08–12.*
