# Session notes — 09 Y crop pair @ 1800 dpi

- Date: 2026-08-03
- Vendor app: SilverFast 9
- USB: `07b3:1825`, vendor driver
- Outcome: **ok** — first feed fixed; second feed tracks Y crop

## Files

| File | Size (approx) | Crop |
|------|---------------|------|
| `09a_crop_top_1800.pcapng` | 66.7 MB / ~49 s | Top of frame |
| `09b_crop_bottom_1800.pcapng` | 66.7 MB / ~52 s | Bottom of frame |

Each file contains **two** full colour pipelines (same feeds twice) — likely two
scans or a double pass in the UI; FEEDL pattern is identical within each file.

## Motor result (the point of this session)

| | Top (`09a`) | Bottom (`09b`) |
|--|-------------|----------------|
| Feed 1 | **28292** | **28292** |
| Feed 2 | **13128** | **20232** |
| Image `0x02` | `0x30` (`AGOHOME`) | `0x30` |
| Image `FEEDL` | 1 | 1 |

- Feed 1 is a **per-model constant** (matches sessions 03–06, 08).
- Feed 2 is the **Y start line** in ASIC steps: bottom crop feeds **7104** steps
  further than top (`20232 - 13128`).
- Session 04 full-ish scan used feed 2 = **13704** (between these two).
- Session 03 / 08 preview used **13128** (same as this top crop).

So plusteklib must **not** hardcode only `13704` forever; positioning is:

```text
feed(28292)           # reference, always
feed(y_start_steps)   # crop-dependent
scan with AGOHOME     # FEEDL=1, 0x02=0x30
```

Exact mm ↔ steps still needs a measured crop height; the **delta** between top
and bottom here is the first hard Y calibration point (7104 steps).

## Geometry at image arm (both passes, both files)

| Register | Top | Bottom |
|----------|-----|--------|
| `DPISET` | 300 (= 1800/6) | 300 |
| `LINCNT` (image) | 3700 | 3700 |
| `STRPIXEL` | 266 | 242 |
| `ENDPIXEL` | 10610 | 10586 |
| span | **10344** | **10344** |

Horizontal span identical; slight STR/END shift (user may have nudged X). Same
output line count → same crop height, different Y origin — exactly what we
wanted.

## Implications for plusteklib

- Keep `feed_to_reference_steps = 28292`.
- Replace single `feed_to_scan_steps = 13704` with a crop/Y-derived value (or
  keep 13704 only as the “default full-frame” constant from session 04 until
  geometry maps mm → steps).
- Do not use GL845-style `geometry.starty` from `y_offset_ta_mm` (that caused
  grinding).
