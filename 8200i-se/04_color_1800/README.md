# 04 — Color scan @ 1800 dpi

## What this is for

Full **color transparency** scan path: configure DPI/exposure/window → start
scan → **bulk IN** image data. Primary oracle for the first working SE scan.

Suggested file: `04_color_1800.pcapng`

Prefer **1800 dpi** (or the vendor’s mid default). Keep the area small if the UI allows.

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| Wireshark + USBPcap | Capture (files can get large) |
| SilverFast / Plustek / VueScan | Perform color scan |
| Empty holder / throwaway strip | No need for precious film |
| Vendor driver | Required |

## How to capture

1. Device ready in vendor app; film holder inserted (empty or scrap is fine).
2. Set **color** (not IR), **~1800 dpi**, smallest useful crop.
3. Start capture.
4. Start the scan; wait until the image appears in the UI.
5. **Stop capture promptly** (bulk traffic balloons the file).
6. Save `04_color_1800.pcapng`; in `NOTES.md` record DPI, bit depth if shown, crop size, duration.

### Filters

```text
usb.idVendor == 0x07b3 && usb.idProduct == 0x1825
usb.transfer_type == 0x03 && usb.idVendor == 0x07b3
```

## Done when

- Control phase configures the scan, then a long **bulk IN** stretch
- Approximate size is plausible for `lines × pixels × 3 × 2` (16-bit RGB)
- Notes record exact DPI/mode

## Priority

Part of the **minimum viable set** with `01` and `02`.
