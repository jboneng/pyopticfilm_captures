# 01 — Enumerate

## What this is for

Capture USB **descriptors and configuration** when Windows first sees the SE:
VID/PID, `bcdDevice`, interfaces, bulk endpoints, max packet sizes.

Feeds: `plusteklib/usb/device.py` (already allows `1825`; confirms endpoints /
`bcdDevice` for GL128).

Suggested file: `01_enumerate.pcapng`

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| Wireshark | Capture + save `.pcapng` |
| USBPcap | USB capture source in Wireshark |
| Device Manager | Confirm Hardware Id `VID_07B3&PID_1825` |

Vendor scan software is **not** required for this session.

## How to capture

1. Install Wireshark + USBPcap; reboot if the USBPcap installer asks.
2. Unplug the scanner.
3. In Wireshark, select a `USBPcapN` interface (may take one try to find the right host controller).
4. **Start** capture.
5. Plug the scanner in; wait until Device Manager shows it.
6. **Stop** capture; save as `01_enumerate.pcapng` in this folder.
7. Fill `NOTES.md` (Windows build, which USBPcap interface worked, `bcdDevice` if visible).

### Display filters (viewing)

```text
usb.idVendor == 0x07b3
usb.idProduct == 0x1825
```

## Done when

- Device appears as `07b3:1825`
- Configuration / interface / bulk IN+OUT endpoints are visible
- File is small and not only hub chatter
