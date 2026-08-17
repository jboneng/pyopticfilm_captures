# SE calibration (AFE + ASIC shading) — capture decode

**Status:** Steps 1–3 complete (offline decode + stationary AFE + ASIC shading
upload). Host dark/white stretch is skipped when ASIC shading is active.

Sources: sessions [`03_home_or_preview`](03_home_or_preview/),
[`04_color_1800`](04_color_1800/).  
Fixtures: [`decoded/afe_shading_fixture.json`](decoded/afe_shading_fixture.json).  
Extractor: `scripts/extract_se_afe_shading.py`.  
Algorithm: `plusteklib/scan/calib_gl128.py` · HW: `Gl128.search_afe`,
`run_asic_shading`.

---

## Two stages (SilverFast)

1. **AFE offset/gain search** — FE indices `0x02`–`0x04` (offsets), `0x05`–`0x07`
   (gains), 16-bit via `0x51`/`0x5d`/`0x5e`.
2. **ASIC shading table** at AHB `0x10014000` — per-pixel
   `(dark, white) × R,G,B` (12 bytes/pixel), declared size `12*N + 4`.

`Calibrator.run` on GL128 now does (1) then (2). Image passes keep `DVDSET` so
the ASIC can apply the table; host `apply_host_calib` is skipped when the cache
entry is marked `asic_shading`.

---

## AFE choreography (session 03 timeline)

Both 03 and 04 use **52** FE writes and the same strip sizes (`3072`, `62268`).

| Order | Action | Notes |
|-------|--------|--------|
| 1 | FE offsets/gains → 0 | init |
| 2 | Gains → `0x80`, strip **3072** | mid probe |
| 3 | Gains → `0xFF`, strip **3072** | high probe |
| 4 | Gains → high (~`0x0140`–`0x016a`) | unit-specific |
| 5 | Wide read **62268** | offset measurement |
| 6 | Offsets → coarse (~36/26/33) | unit-specific |
| 7–8 | Gains re-probe `0x80` / `0xFF` + strips | |
| 9–10 | Gains mid-high → low → settle | |
| 11 | Shading upload #1 | dark + white=`0x2000` |
| 12 | Gains settle | |
| 13 | Shading upload #2 | dark + measured white |
| 14 | Offsets fine tweak | |

Each strip acquire uses the usual start recipe (`0x0d` → `0x01=0x03` → `0x0f`)
with **no motor** on the first shading-related passes (`0x02=0x00` in session 04
notes). AFE window: `STRPIXEL=64`, `ENDPIXEL=576` (512 px).

### AFE decision rule (Step 2)

- Exact SilverFast closed-form mean→FE update was **not** recovered.
- Runtime: SANE-style dichotomy (`search_afe`) toward dark/white targets.
- Best-effort coarse offsets: `round((65535 - wide_mean) / 1155)`.

---

## Shading table layout

```text
per pixel (LE u16):  dark_r white_r  dark_g white_g  dark_b white_b
upload size:         12*N + 4 zero pad
AHB address:         0x10014000
measure acquire:     128 lines × window_width × RGB16
```

| Session | Declared size | Table N | Acquire width | First upload white |
|---------|---------------|---------|---------------|--------------------|
| 03 preview (1200) | 21064 | 1755 | 1728 | all `0x2000` |
| 04 @ 1800 | 30208 | 2517 | 2478 | all `0x2000` |
| 06 @ 3600 | 60416 | ~5034 | (window) | all `0x2000` |
| 13 ladder | see NOTES | per-PPI | (crop-dependent) |

Measure bulk sizes match `128 × window_width × 6` (session 04: 1 903 104 =
128×2478×6). First upload
broadcasts a constant per-channel dark; second keeps that dark and replaces
white with per-pixel averages. Both library uploads stay **stationary**
(`0x02=0x00`); capture’s second pass used `MTRPWR` only — we do not arm motor.

`SHADING_WIDTH_BY_DPI` covers every SilverFast PPI (session 13); 1200/1800/3600
prefer the fuller-frame sessions above where noted.

### Shading window = image window

Both shading passes run through **the same `STRPIXEL`/`ENDPIXEL`/`DPISET` as the
image pass that follows** — not a fixed calibration window. Sessions 04 and 05
at 1800 dpi:

```text
shading pass 1/2 : STRPIXEL=578 ENDPIXEL=10490 DPISET=300 LINCNT=128  (0x02=0x00 / 0x10)
image pass       : STRPIXEL=578 ENDPIXEL=10490 DPISET=300 LINCNT=6628 (0x01=0x23)
```

Regenerate with `scripts/dump_se_shading_windows.py`. The ASIC indexes the table
by acquired pixel, so measuring through a different window or `DPISET` and then
scanning with `DVDSET` on drops columns periodically instead of flattening them
(`Gl128._shading_window` keeps the two in step).

---

## Library helpers

| Symbol | Role |
|--------|------|
| `AFE_SEARCH_PHASES_CAPTURE` | Documented phase skeleton |
| `AfeFrontend` / `AfeSearchConfig` | FE state + dichotomy targets |
| `run_afe_dichotomy` / `search_afe_codes` | Pure AFE search |
| `shading_width_for_resolution` / `declared_shading_size` | N and AHB size |
| `average_rgb16_columns` / `build_measured_shading_table` | Strip → blob |
| `pack_shading_table` / `make_unity_white_table` | AHB codec |
| `Gl128.acquire_afe_strip` / `search_afe` | Stationary AFE |
| `Gl128.acquire_shading_strip` / `run_asic_shading` | Stationary shading |

---

## IR shading (session 05)

SilverFast IR uploads use **zero dark terms** and near-equal channel whites.
Lab measures the same way (dark diagnostic → zero table dark; white under IR
LED; `equalize_ir_white_columns`), but **does not arm ASIC DVDSET for IR** —
live DVDSET clipped the image to ~99% `0xFFFF`.

When `validate_ir_shading_table` accepts the white profile (mean ≥ 10000, no
bars), Lab stores `last_ir_host_white` for diagnostics / future app use.
**Scan output is the raw CCD frame only** (`rgb` → path from the TIFF text box):
no host `flatten_ir_*` / `enhance_ir_defect_contrast` and no automatic `*_IR.tif`
sidecar. Lab IR-mode **preview** applies `mild_ir_ghost_fade` (soft ghost
compress + mild defect deepen); the written TIFF stays raw. Aggressive
`enhance_ir_defect_contrast` remains opt-in in `calib_gl128.py` for apps.
iSRD post-process belongs in the application on top of plusteklib.

Pegged IR AFE: if strip means are already healthy (≥ 8000), keep max gains and
session-05 offsets `(6,16,12)`; low session-05 gains only when still dim.

## Next (HW validation)

Lab IR path (after park / power-cycle, hand on power):

1. Stationary AFE + white measure with motor **disarmed**, then arm for image.
2. IR-only scan — DVDSET off; single TIFF is the illuminated CCD frame (no
   `*_IR` sidecar). Lab preview may show `mild_ir_ghost_fade`; file stays raw.
3. Colour / Prescan still AFE-only (DVDSET off).
4. Two-pass: colour image → disarm → IR measure → re-arm → IR image (optional
   green plane on `ScanImage.ir` for apps; not auto-written).

Motor-gated unit tests cover refuse-while-armed; live flatness needs this smoke.
