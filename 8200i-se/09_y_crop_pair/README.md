# 09 — Same DPI, two different Y crops

## Status: **done** (2026-08-03)

Confirmed: feed 1 stays **28292**; only feed 2 changes with vertical crop
(**13128** top, **20232** bottom). See [`NOTES.md`](NOTES.md) and
[`MOTOR.md`](../MOTOR.md).

## Captured files

| File | Crop |
|------|------|
| `09a_crop_top_1800.pcapng` | Top of frame @ 1800 dpi colour |
| `09b_crop_bottom_1800.pcapng` | Bottom of frame @ 1800 dpi colour |

## Result (short)

```text
always:     feed(28292)
then:       feed(y_start_steps)   # 13128 top, 13704 full-ish, 20232 bottom
then:       image with AGOHOME, FEEDL=1
```

## Original how-to (historical)

Two separate captures at **1800 dpi**, colour, no IR — only the Y crop changed.
