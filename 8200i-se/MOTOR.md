# SE motor sequences (capture-derived)

**Status:** decoded from sessions 03–06. The previous plusteklib
`feed()` / `home()` path was **not** faithful and caused grinding; motor
moves stay gated in code until a careful hardware re-test.

Source timelines: `03_home_or_preview/timeline.txt`, register writes in
`decoded/se_extract.json`.

---

## What SilverFast always does (from home)

Every full colour / IR scan (sessions 04, 05, 06) and the Y-crop pair (session
09) use the same **first** feed; only the **second** feed tracks vertical start:

| Step | `FEEDL` | `0x02` | Notes |
|------|---------|--------|-------|
| 1. Fast reposition | **28292** | `0x18` → after done `0x08` | Constant in every session |
| 2. Scan-start feed | **crop-dependent** | `0x18` | 13128 top / preview; 13704 full-ish (04); **20232** bottom (09b). Library: `feed_to_scan_steps_for_area()` |
| 3. Image acquire | **1** | **`0x30`** (`AGOHOME` \| `MTRPWR`) | Carriage parks as scan ends; USB depth 8-bit (`0x33=0x1F`, `0xAF=0xFF`) |

There is **no** standalone `FEEDL=0` home seek in the captures. Return-to-home
is a side-effect of step 3.

## Why the old path ground

| Old plusteklib | Captures |
|----------------|----------|
| Pre-scan feed of `starty≈8078` from GL845 `y_offset_ta_mm` | Never used; real feeds are 28292 + 13704 |
| Short recipe: `0x02=0x18`, FEEDL, `0x0d=0x07`, `0x0f=0x01` | Fuller setup + slope upload; feed START is **`0x0f` only** (no `0x0d`) |
| Invented `home()` with `FEEDL=0` + `AGOHOME\|MTRPWR\|FASTFED` | Home is `0x02=0x30` on the **image** pass |

## Fast-feed recipe (session 03, t≈8.98 → 9.97)

Minimal ordered replay before `0x0f = 0x01`:

1. `0x01 = 0x22` (SCAN clear)
2. `0x04 = 0x42`, `0x05 = 0x48`
3. `FEEDL` = steps (24-bit BE at `0x3d`)
4. `0xa6`–`0xa9` = 0
5. exposure `0x7d`–`0x7f` = 14000 (`0x0036b0`)
6. fixed window regs seen in capture (`0x80`–`0x87`, `DPISET=200`, `0x1c`/`0x1d`, `0xa4`/`0xa5`/`0xaa`/`0xab`)
7. `0x02 = 0x18` (`MTRPWR` \| `FASTFED`)
8. `0xae = 0`, `0xaf = 0x7f`
9. Upload **fast** slope table (512 B) to `0x1000c000` and `0x10010000`
10. `0x0f = 0x01` — **do not** write `0x0d` here
11. Poll vendor probe **`wIndex=0x21`** until value `0x04` (~1 s for 28292)
12. `0x02 = 0x08`, then `FEEDL = 1`

Second feed (13704) is the same shape; completion is visible on `0x101`
(`0xd5` while moving → `0xf5`/`0xf4` idle).

Slope table in `plusteklib/device/tables_8200i_se.py` is the **fast** ramp
(`0x16de, 0x06db, …`). The slow ramp (`0x1fb4, …`) is for shading passes only.

## Safe software policy

- Do not command motor until unit tests assert this register order against a
  fake transport, **and** the user explicitly re-enables motor for a hardware
  ladder (hand on power plug).
- Fast-feed completion (`wIndex=0x21 -> 0x04`) can be **stale** across feed
  pairs. Only accept `0x21=0x04` as “this feed finished” after you have
  observed motion on the `0x101` status register (`MOTORENB` set at least
  once). Do not let a stale `0x21=0x04` truncate positioning feeds.
- `home()` must not invent a reverse seek; if not at home, refuse and tell the
  operator to park with SilverFast / power-cycle.
- Scan positioning uses the capture constants above, not `geometry.starty`.
- If an `AGOHOME` park times out, refuse subsequent SE scans/positioning until
  the carriage is parked again with SilverFast or the scanner is power-cycled.
- **Scan-window end (the real limit):** `LINCNT` is in ASIC-dpi units, four per
  output line, so travel is `LINCNT × 25.4 / (4 × asic_dpi)` — see
  [`decoded/ppi_lincnt_feed.json`](decoded/ppi_lincnt_feed.json). Feed steps are
  14400/inch, and every capture satisfies:

  ```text
  feed2 + travel_steps <= 27636          # 48.75 mm
  max_lincnt(feed2, dpi) = (27636 - feed2) * asic_dpi / 3600
  ```

  | Session | feed2 | LINCNT | dpi | travel mm | ends at |
  |---------|-------|--------|-----|-----------|---------|
  | 03 preview / 09a | 13128 | 4836 | 1200 | 25.6 | **27636** |
  | 13 PPI ladder | 13560 | 2292…27476 | 150…7200 | 24.2 | ~27300 |
  | 04 colour | 13704 | 6628 | 1800 | 23.4 | 26960 |
  | 09b bottom | 20232 | 3700 | 1800 | 13.1 | **27632** |

  Sessions 03 and 09b land on the stop to within 4 steps, which is what makes
  27636 a mechanical limit rather than a user crop. ``max_lincnt_for`` floors
  the step product to a multiple of ``image_lincnt_per_line`` (4) so USB row
  count and pair averaging stay aligned — that matches both captures
  (13128@1200 → 4836; 20232@1800 → 3700). Without the floor, odd values such as
  5803 at 1440/full-window scramble the image buffer.

  The absolute mm scale is pinned by the film: the ladder crop is 36.06 ×
  24.24 mm, a 3:2 35 mm frame, and the whole window is 25.59 mm — one frame plus
  ~0.8 mm of holder at each end. (Steps and LINCNT are self-consistent at any
  scale, so the frame is what fixes it.)

  Lab ground because the old native-units math turned a 25 mm request at 1200
  dpi into `LINCNT=7086` — 37.5 mm of travel from 13128, ending 12 mm past the
  stop. `Gl128ScanSession` now refuses any LINCNT whose travel overruns it.

## Follow-up capture folders

| Folder | Question |
|--------|----------|
| [08_midtravel_home/](08_midtravel_home/) | **Done** — Cancel during preview image: `0x03` strobe + `0x01=0x22`, park via `AGOHOME` (no standalone reverse-home) |
| [09_y_crop_pair/](09_y_crop_pair/) | **Done** — feed1=28292 always; feed2 tracks Y (13128 top, 20232 bottom) |
| [10_back_to_back/](10_back_to_back/) | **Done** — next scan always full feeds from home after `AGOHOME` park |
| [11_bit_depth_1800/](11_bit_depth_1800/) | **Done** — depth is `0x33`/`0xAF`; HDR UI ≠ 16-bit USB |
| [12_abort_during_feed/](12_abort_during_feed/) | **Blocked** in SilverFast 9 — evidence: `12_fast_cancel_feeds_complete.pcapng` |
