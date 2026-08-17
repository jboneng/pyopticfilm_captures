# Session notes — 13 PPI ladder

- Date: 2026-08-04
- Vendor app + version: SilverFast 9
- USB: `07b3:1825`, vendor driver
- Outcome: **ok** — full SilverFast PPI set captured and decoded

## SilverFast resolution list

```text
150, 300, 600, 720, 900, 1200, 1440, 1800, 2400, 3600, 7200
```

All are divisors of 7200. No free-entry values beyond this preset list were needed.

## Captured files

| File | UI PPI | Size (approx) |
|------|--------|---------------|
| `13_150ppi_color.pcapng` | 150 | 15 MB |
| `13_300ppi_color.pcapng` | 300 | 15 MB |
| `13_600ppi_color.pcapng` | 600 | 15 MB |
| `13_720ppi_color.pcapng` | 720 | 21 MB |
| `13_900ppi_color.pcapng` | 900 | 32 MB |
| `13_1200ppi_color.pcapng` | 1200 | 53 MB |
| `13_1440ppi_color.pcapng` | 1440 | 75 MB |
| `13_1800ppi_color.pcapng` | 1800 | 115 MB |
| `13_2400ppi_color.pcapng` | 2400 | 200 MB |
| `13_3600ppi_color.pcapng` | 3600 | 440 MB |
| `13_7200ppi_color.pcapng` | 7200 | 942 MB |

Decode extract: [`decoded_ppi_ladder.json`](decoded_ppi_ladder.json).
Feed + LINCNT extract: [`../decoded/ppi_lincnt_feed.json`](../decoded/ppi_lincnt_feed.json)
(regenerate from the [pyopticfilm](https://github.com/jboneng/pyopticfilm) repo:
`scripts/extract_se_feeds.py`).

## Image-pass LINCNT + feeds (session 13)

Feed1 is always **28292**. Feed2 is **~13560** for this ladder’s crop (not
the preview-top **13128** / session-04 **13704**).

**LINCNT is in ASIC-dpi units, four per output line** — not native 7200 dpi
travel. That is why it tracks PPI here: this ladder is **one fixed 24.2 mm
crop** captured eleven times. `LINCNT / dpi` is 3.816 in every row, each bulk
buffer holds exactly `LINCNT / 2` rows, and those rows are Y sampled at *twice*
the programmed dpi, so pairs average into `LINCNT / 4` output lines.

The width settles the factor: the crop is 10224 optical columns = 36.06 mm, so
`36.06 × 24.24` is a 3:2 35 mm frame. Calling the buffer rows output lines would
make it 36 × 48.5 mm and stretch every scan 2× vertically.

```text
travel_mm = LINCNT * 25.4 / (4 * asic_dpi)      # asic_dpi floors at 600
```

| PPI | feed1 | feed2 | LINCNT | usb rows | lines | travel mm | ends at (steps) |
|-----|-------|-------|--------|----------|-------|-----------|-----------------|
| 150 | 28292 | 13560 | 2292 | 1146 | 573 | 24.3 | 27312 |
| 300 | 28292 | 13560 | 2292 | 1146 | 573 | 24.3 | 27312 |
| 600 | 28292 | 13560 | 2292 | 1146 | 573 | 24.3 | 27312 |
| 720 | 28292 | ~13548 | 2748 | 1374 | 687 | 24.2 | 27288 |
| 900 | 28292 | 13560 | 3436 | 1718 | 859 | 24.2 | 27304 |
| 1200 | 28292 | 13560 | 4580 | 2290 | 1145 | 24.2 | 27300 |
| 1440 | 28292 | ~13558 | 5496 | 2748 | 1374 | 24.2 | 27298 |
| 1800 | 28292 | 13560 | 6868 | 3434 | 1717 | 24.2 | 27296 |
| 2400 | 28292 | 13560 | 9156 | 4578 | 2289 | 24.2 | 27294 |
| 3600 | 28292 | 13560 | 13732 | 6866 | 3433 | 24.2 | 27292 |
| 7200 | 28292 | 13560 | 27476 | 13738 | 6869 | 24.2 | 27298 |

`usb rows` is checked against `bulk_size / (output_pixels * 3 * 2)` in every
row — the buffer is `LINCNT/2` rows at all eleven PPI, including 7200 where
oversample is 1.

Cross-check sessions (same extractor):

| Session | feed2 | LINCNT | dpi | travel mm | ends at |
|---------|-------|--------|-----|-----------|---------|
| 03 preview | **13128** | **4836** | 1200 | **25.6** | **27636** |
| 04 colour | **13704** | **6628** | 1800 | 23.4 | 26960 |
| 09a top | **13128** | 3700 | 1800 | 13.1 | 20528 |
| 09b bottom | **20232** | 3700 | 1800 | 13.1 | **27632** |

Session 03 covers the whole 25.6 mm window — one frame plus a little holder,
which is the chrome visible above and below the image in a preview. 09a and 09b
are its 13.1 mm halves, offset by 7104 steps (12.5 mm), so they overlap by half
a millimetre instead of leaving a gap.

**Motor safety:** the limit is where the pass *ends*, not LINCNT itself. No
capture passes **27636** steps (14400 steps/inch) and two land on it, so that is
the scan-window stop. The Lab grind asked for 25 mm at 1200 dpi, which the old
native-units formula turned into `LINCNT=7086` = 37.5 mm of travel, ending 12 mm
past the stop.

## Image-pass register table (decoded)

| PPI | DPISET | dpi/6 | LPERIOD | 0x2B | 0xA5/AB | shading N | STAGGER |
|-----|--------|-------|---------|------|---------|-----------|---------|
| 150 | **100** | 25 | 11064 | 1 | 2 | 865 | no |
| 300 | **100** | 50 | 11064 | 1 | 2 | 865 | no |
| 600 | 100 | 100 | 11064 | 1 | 2 | 865 | no |
| 720 | 120 | 120 | 11106 | 1 | 2 | 1037 | no |
| 900 | 150 | 150 | 11170 | 1 | 2 | 1297 | no |
| 1200 | 200 | 200 | 11277 | 2 | 2 | 1730 | no |
| 1440 | 240 | 240 | 11362 | 2 | 2 | 2075 | no |
| 1800 | 300 | 300 | 11490 | 2 | 2 | 2595 | no |
| 2400 | 400 | 400 | 11703 | 3 | 1 | 3461 | no |
| 3600 | 600 | 600 | 13407 | 4 | 1 | 5192 | no |
| 7200 | 1200 | 1200 | 15963 | 0x17 | 1 | 10385 | **no** |

### Findings

1. **DPISET = dpi/6** for PPI ≥ 600. Below that, SilverFast programs **DPISET=100** (same as 600) — 150/300/600 share byte-identical ASIC image config; host downsamples.
2. **STAGGER never set**, including at **7200**.
3. Shading N tracks crop width (~`width × 65/64`); values above are for this ladder’s crop.
4. Wired into `Model8200iSE` + `SHADING_WIDTH_BY_DPI` (1200/1800/3600 N prefer fuller sessions 03/04/06 where larger).
5. Feeds are **PPI-independent**; only LINCNT / DPISET / window regs track resolution.
