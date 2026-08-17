# 08 — Preview cancel → park via AGOHOME

## Status: **done** (2026-08-03)

SilverFast does **not** expose a standalone reverse-home. Cancel during a
prescan/preview only takes effect after positioning feeds finish and the image
pass has started; the head then parks because `AGOHOME` (`0x02 = 0x30`) was
already armed. See [`NOTES.md`](NOTES.md) and [`MOTOR.md`](../MOTOR.md).

## Captured files

| File | Contents |
|------|----------|
| `08a_preview_cancel_x3.pcapng` | Three preview cycles, Cancel during image |
| `08b_preview_cancel.pcapng` | One cycle, Cancel ~0.3 s into image bulk |
| `08c_preview_cancel.pcapng` | Same as `08b` (repeat) |

Fast Cancel-as-soon-as-possible (feeds still completed) lives under session 12:
[`12_fast_cancel_feeds_complete.pcapng`](../12_abort_during_feed/12_fast_cancel_feeds_complete.pcapng).

## What we learned

1. Preview pipeline: `FEEDL=28292` → `FEEDL=13128` → image with `0x02=0x30`.
2. Cancel recipe on `0x03` / `0x01` (byte-identical in `08b`/`08c`):

   ```text
   0x03=0x30, 0x20;  0x01=0x22;  0x03=0x10, 0x00, 0x20, 0x30, 0x20, 0x30
   ```

   No new `FEEDL`, no change to `0x02`.
3. Status after cancel: `0x101` → `0xa5` → `0xad` → `0xec` while `AGOHOME` parks.
4. There is **no** separate reverse-home register sequence in these traces.

## Implications for plusteklib

- Set `AGOHOME` before image `START`.
- Image-phase abort: clear `SCAN` (+ lamp handling like the capture); do not
  invent a standalone home seek.
- Mid-feed abort: not available in SilverFast — see [session 12](../12_abort_during_feed/).

## Original goal (historical)

This folder was opened to prove a mid-travel reverse/home recipe. That path does
not exist in SilverFast 9 for preview Cancel; park is a side-effect of
`AGOHOME` on the image pass.
