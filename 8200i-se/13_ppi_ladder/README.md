# 13 — PPI / resolution ladder

## Status: **done** (2026-08-04)

Full SilverFast 9 PPI set for the 8200i SE:

`150, 300, 600, 720, 900, 1200, 1440, 1800, 2400, 3600, 7200`

See [`NOTES.md`](NOTES.md) and [`decoded_ppi_ladder.json`](decoded_ppi_ladder.json).

## Headline results

- `DPISET = dpi/6` at ≥600; **floors at 100** for 150/300 (ASIC = 600 path).
- Per-DPI `LPERIOD`, `0x2B`, `0xA5`/`0xAB` captured for every preset.
- **`STAGGER` clear at 7200** (and every other PPI).
- Library: `Model8200iSE.resolutions_dpi` now lists the full set.

## Captured files

| File | PPI |
|------|-----|
| `13_150ppi_color.pcapng` … `13_7200ppi_color.pcapng` | one colour scan each |
