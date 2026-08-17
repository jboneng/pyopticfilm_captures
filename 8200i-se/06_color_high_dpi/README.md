# 06 — Color high DPI (3600 or 7200)

## What this is for

Optional **higher-resolution color** scan to reveal DPISET, stagger, exposure,
and memory-layout differences vs 1800.

Suggested files:

- `06_color_3600.pcapng` and/or
- `06_color_7200.pcapng`

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| Wireshark + USBPcap | Capture (7200 files are large — keep crop tiny) |
| Vendor app | High-DPI color scan |
| Vendor driver | Required |

## How to capture

1. Prefer finishing `04` first.
2. Start capture → one short **3600** or **7200** color scan → stop ASAP.
3. Save with DPI in the filename; note crop and DPI in `NOTES.md`.

## Done when

- Bulk image traffic present
- DPI clearly documented (UI setting + filename)

## Priority

Optional. Skip until 1800 color path is decoded if time is limited.
