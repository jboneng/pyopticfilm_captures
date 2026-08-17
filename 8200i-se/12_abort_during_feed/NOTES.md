# Session notes — 12 abort during feed

- Date: 2026-08-03
- Vendor app: SilverFast 9
- USB: `07b3:1825`, vendor driver (not WinUSB)
- Outcome: **blocked** — no mid-feed abort in this UI

## Evidence file

| File | Size (approx) | What it shows |
|------|---------------|----------------|
| `12_fast_cancel_feeds_complete.pcapng` | 3.5 MB / ~13 s | Prescan, Cancel pressed as fast as possible |

### Decode summary (`12_fast_cancel_feeds_complete`)

| Step | Result |
|------|--------|
| Feed 1 `28292` | Ran to completion (`0x02=0x18` → `0x08`) |
| Feed 2 `13128` | Ran to completion |
| Image arm | `0x02=0x30` (`AGOHOME`), `0x01=0x23` |
| Image start | `0x0f=0x01` |
| Cancel | Lamp strobe + `0x01=0x22` (~0.3 s into bulk) — same as session 08 |

Cancel did **not** interrupt either feed.

## UI findings

| Path | Cancel behaviour |
|------|------------------|
| Prescan / preview | Enabled, but only honoured after positioning + image start (session 08 + this file) |
| Full scan | Button stays **disabled** for the whole job |

## Conclusion

No mid-feed `stop_motor` recipe from SilverFast 9. Session 08 covers image-phase
abort (`clear SCAN` while `AGOHOME` already set). Do not force with power/USB
yank.

If VueScan or Plustek QuickScan can cancel mid-motion, capture that here later.
