# 11 — 16-bit / 48-bit output @ 1800 dpi

## Status: **done** (2026-08-03)

SilverFast **48 → 24 Bit** vs **48 Bit HDR RAW** — identical USB image depth:
`0x33=0x1F` / `0xAF=0xFF` (8-bit). Shading still uses `0x04` / `0x46` (16-bit).
See [`NOTES.md`](NOTES.md).

## Captured files

| File | SilverFast mode |
|------|-----------------|
| `11a_color_1800_24bit.pcapng` | 48 → 24 Bit |
| `11b_color_1800_48bit.pcapng` | 48 Bit HDR RAW |

## Result (short)

```text
shading:  0x33=0x04, 0xAF=0x46   # 16-bit
image:    0x33=0x1F, 0xAF=0xFF   # 8-bit  (both UI modes)
```
