# 10 — Back-to-back scans (warm reposition)

## Status: **done** (2026-08-03)

After each image, SilverFast parks via `AGOHOME`, then the next scan repeats the
**full** feed pair from home (`28292` + crop feed). No mid-frame short
reposition. See [`NOTES.md`](NOTES.md).

## Captured file

`10_back_to_back_1800.pcapng` (~244 MB) — two UI scans with RGB+IR each → four
identical feed pipelines (app left open between jobs).

## Result (short)

```text
each pass (RGB or IR):
  feed(28292) → feed(y_start) → image+AGOHOME → park at home
next pass / next scan: same again from home
```
