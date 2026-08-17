# 02 — Cold boot / open (init)

## What this is for

Capture the vendor app’s **soft reset + register init blast** (GPO, frontend,
memory layout, clocks) from power-on until the scanner is idle/ready — **before**
any scan.

This is the most important control-transfer capture for filling
`Model8200iSE.init_regs` / GPO / FE / sensor tables.

Suggested file: `02_cold_boot_open.pcapng`

## Tools needed (Windows)

| Tool | Why |
|------|-----|
| Wireshark + USBPcap | Capture |
| SilverFast SE / Plustek app or VueScan | Opens device and runs init |
| Stock **vendor** USB driver | Must not be WinUSB yet |

## How to capture

1. Confirm Device Manager still shows the vendor binding for `07B3:1825`.
2. Power the scanner **fully off**, then on (cold boot).
3. Start Wireshark on the correct `USBPcap` interface.
4. Launch SilverFast (or VueScan); wait until the UI shows the device ready / lamp idle.
5. **Stop before scanning** (no preview, no acquire).
6. Save `02_cold_boot_open.pcapng` here; write `NOTES.md` (app version, cold boot yes/no).

### Useful filters

```text
usb.idVendor == 0x07b3 && usb.idProduct == 0x1825
usb.bmRequestType && usb.idVendor == 0x07b3
```

## Done when

- Many vendor **control** OUT/IN transfers early after open
- Sequence is long enough to cover init (not a single descriptor read)
- No long bulk-IN image stream (that belongs in scan sessions)

## Decode later

Tabulate chronological register writes → drop JSON under `../decoded/`
(see that folder’s README).
