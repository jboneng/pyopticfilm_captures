# 12 — Abort while the head is feeding

## Status: **blocked** in SilverFast 9 (2026-08-03)

No mid-feed abort path was found. Do **not** yank power/USB to force a trace.

| Path | What happens |
|------|----------------|
| Prescan / preview | Cancel works, but only **after** both feeds finish and image bulk starts (session 08). Even Cancel-as-fast-as-possible does not interrupt `FEEDL=28292` / `13128`. |
| Full scan | Cancel button stays **disabled** for the entire job. |

## Evidence file

| File | Role |
|------|------|
| `12_fast_cancel_feeds_complete.pcapng` | Prescan + Cancel as fast as possible; feeds complete; Cancel clears `SCAN` only after image `START` |

Full decode notes: [`NOTES.md`](NOTES.md). Image-phase cancel recipe (the one
SilverFast *does* support): [session 08](../08_midtravel_home/).

## plusteklib policy

- Keep motor moves gated until a deliberate HW re-test.
- Image-phase abort: clear `SCAN` + existing `AGOHOME` (session 08).
- Mid-feed abort: no vendor recipe — if we ever need one, try another app
  (VueScan / Plustek QuickScan) or accept that feeds are atomic.

## If another app can abort mid-feed later

1. Start capture → start job → Cancel **during** the first long feed (before
   `0x02 = 0x30`).
2. Save as `12_abort_during_feed.pcapng` and update `NOTES.md`.

## Original goal

Capture how the vendor kills the motor mid-feed for a faithful `stop_motor()`.
SilverFast 9 does not expose that.
