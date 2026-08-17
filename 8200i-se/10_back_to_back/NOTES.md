# Session notes — 10 back-to-back scans @ 1800 dpi

- Date: 2026-08-03
- Vendor app: SilverFast 9 (colour + IR / iSRD, same crop both times; app left open)
- File: `10_back_to_back_1800.pcapng` (~244 MB, ~140 s)
- Outcome: **ok** — warm second scan still parks home, then repeats the full
  feed pair from home (no mid-frame short reposition)

## What the decode shows

Four identical positioning+image pipelines = **two UI scans × (RGB + IR)**.
Each pass (colour or infrared) uses the same feed pair and parks via `AGOHOME`:

| Cycle | Likely pass | Feed 1 | Feed 2 | `AGOHOME` | Park (`0xa5`→`0xad`) |
|-------|-------------|--------|--------|-----------|----------------------|
| 1 | scan1 RGB | t=9.27 `28292` | t=10.34 `13128` | t=11.64 | t=35.88 |
| 2 | scan1 IR | t=42.41 `28292` | t=43.49 `13128` | t=44.79 | t=69.03 |
| 3 | scan2 RGB | t=80.92 `28292` | t=81.99 `13128` | t=83.30 | t=107.53 |
| 4 | scan2 IR | t=114.05 `28292` | t=115.11 `13128` | t=116.41 | t=140.64 |

Feed 2 = **13128** (same as session 03/08/09a top/preview crop).

## Warm reposition conclusion

Between cycles, status reaches **home** (`0x101` with home bit, `0xad`/`0xec`),
then SilverFast does a short re-init (`0x02=0x78`) and the usual calib block,
then **again** `feed(28292)` + `feed(13128)`.

There is **no** shorter “already mid-frame, just nudge Y” feed for the IR
pass or for scan #2. Each channel pass starts from home with the full pair.
plusteklib can treat every pass as: require home → `28292` → crop feed →
image with `AGOHOME`.

## Gap example (cycle 1 RGB → cycle 2 IR)

1. Image parks: `0xa5` → `0xad` (home) → `0xec` (~35.9 s)
2. Re-init writes `0x01=0x22`, `0x02=0x78` (~36.5 s) while still home
3. Calib / shading activity leaves home briefly (~39.3 s)
4. Full feed pair starts again at ~42.4 s

## Implications for plusteklib

- `position_for_full_frame_scan()` from home every time is capture-faithful —
  including RGB→IR within one job.
- Do not optimise a warm path until a capture shows one.
- `AGOHOME` on each image/IR pass is what returns home between passes.
