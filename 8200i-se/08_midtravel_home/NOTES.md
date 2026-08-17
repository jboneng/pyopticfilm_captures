# Session notes — 08 preview cancel / AGOHOME park

- Date: 2026-08-03
- Vendor app: SilverFast 9 (prescan / preview)
- USB: `07b3:1825`, vendor driver (not WinUSB)
- Outcome: **ok** — Cancel-during-image recipe decoded; no standalone reverse-home

## Files

| File | Size (approx) | What it is |
|------|---------------|------------|
| `08a_preview_cancel_x3.pcapng` | 25.9 MB / ~50 s | Three full preview cycles; Cancel during image on each |
| `08b_preview_cancel.pcapng` | 3.5 MB / ~13 s | One cycle; Cancel ~0.3 s into image bulk |
| `08c_preview_cancel.pcapng` | 3.5 MB / ~13 s | Repeat of `08b` |

Related (session 12): `12_fast_cancel_feeds_complete.pcapng` — Cancel pressed
as fast as possible after Prescan; both feeds still completed before Cancel
took effect.

## Per-cycle motor pattern (`08b` / `08c`)

| Step | Detail |
|------|--------|
| Feed 1 | `FEEDL=28292`, `0x02=0x18` → done with `0x02=0x08` |
| Feed 2 | `FEEDL=13128`, `0x02=0x18` |
| Image arm | `0x02=0x30` (`AGOHOME`), `FEEDL=1`, `0x01=0x23` |
| Image start | `0x0f=0x01`, bulk begins |
| Cancel | Lamp strobe on `0x03`, then `0x01=0x22` (clear `SCAN`), then finish strobe |
| Park | `0x101`: `0x81`/`0x85` → `0xa5` → `0xad` → `0xec` |

No writes to `FEEDL` or `0x02` after Cancel. Park uses the already-armed
`AGOHOME`.

`08a` shows the same pattern three times (second feed always `13128`, preview
crop — not the full-scan `13704`).

## Cancel recipe (stable across `08b` / `08c`; matches session 03 return-home)

Exact register writes (decoded from `08b`/`08c`):

```text
0x03 = 0x30
0x03 = 0x20
0x01 = 0x22          # clear SCAN
0x03 = 0x10
0x03 = 0x00
0x03 = 0x20
0x03 = 0x30
0x03 = 0x20
0x03 = 0x30          # lamp ends ON
```

Then:

1. **No** new `FEEDL`, **no** change to `0x02`
2. Status `0x101` walks through motor-active then idle/home (`0xa5` → `0xad` → `0xec`)

plusteklib: `Gl128.stop_motor` replays this when `AGOHOME` is armed.
## What this does not prove

- Abort **during** `28292` / `13128` (before `AGOHOME`) — SilverFast never did
  that here; see session 12.
- A reverse-home with the carriage mid-frame and `AGOHOME` clear.

## Implications for plusteklib

- Preview/image: set `AGOHOME` before `START`.
- Abort during image: replay capture lamp strobe + clear `SCAN`; do not
  invent a standalone home seek.
- Mid-feed `stop_motor`: unknown / unavailable in this UI.
