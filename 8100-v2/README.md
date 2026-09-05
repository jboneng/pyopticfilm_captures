# Plustek OpticFilm 8100 V2 — capture methodology

This directory contains USB captures of a Plustek OpticFilm 8100 V2 using the vendor
Windows driver / SilverFast 9, following the same capture methodology as the
[8200i-se](../8200i-se/) dataset in this repo. See [SESSIONS.md](SESSIONS.md) for the
session index.

This README describes how to capture and organize the data. It is not a protocol
specification — register meaning, shared tables, or parameter relationships must be
established from the captures themselves and from `pyopticfilm` driver evidence.

## Device

| Field | Value |
|-------|-------|
| Model | Plustek OpticFilm 8100 V2 |
| USB | `07b3:1824` |
| ASIC | Genesys GL128 |
| Capture application | SilverFast 9 / vendor Windows driver |
| Host OS | Windows 11 |
| Film | 35 mm negative |
| Capture method | Wireshark + USBPcap |

Do not switch the scanner to WinUSB for these reference captures — SilverFast needs the
vendor driver bound to the device.

Wireshark display filter:

```text
usb.idVendor == 0x07b3 && usb.idProduct == 0x1824
```

## Global capture settings

Use these unless a session explicitly requires something different.

1. Open Wireshark → **Capture Options**.
2. Select the `\\.\USBPcap*` interface used by the scanner. Unplug/replug can help
   identify the correct one if more than one is listed.
3. **Snap length: 65535.** Required to preserve full USB bulk transfer contents.
4. Leave capture **buffer size** at default for small sessions (01–03). For the large
   image-transfer sessions (04–06, each 400+ MB of bulk-IN traffic), raise the USBPcap
   kernel buffer size before capturing to reduce dropped packets — Capture Options →
   the `\\.\USBPcap*` interface → gear/edit icon → buffer size — try 64 MiB or higher if
   you see dropped-packet warnings.
5. Disable "Use pcap-ng format" is NOT needed — save as `.pcapng` (default).
6. Disable "Use network protocol filters in capture options".
7. Perform exactly **one defined action per capture**. Start the capture *before*
   performing the action, and stop it only after the scanner has finished and the
   carriage has returned home/settled.
8. Minimize unrelated USB activity on the bus while capturing.
9. Record environmental details in that session's `NOTES.md` immediately, while
   fresh.

## Physical safety

Keep the scanner's power switch / power connection accessible.

If the carriage grinds, chatters, stalls, or otherwise behaves abnormally:

1. Power off.
2. Unplug the power cord.
3. Wait a few seconds.
4. Reconnect power, power on.
5. Re-home using SilverFast.
6. Record what happened in the session's `NOTES.md`.
7. Re-capture if the recovery itself is relevant evidence.

Do not force the carriage by hand. Do not cancel an image transfer mid-scan unless a
known-safe procedure has been established first.

## Film / SilverFast consistency

For sessions that involve scanning:

- keep the same 35 mm negative loaded, in the same frame/holder position, across
  comparable sessions
- Color mode, iSRD **off**, HDR **off**, plain output, unless a session says otherwise
- do not move the holder between sessions meant to be compared

Consistency matters because the goal is to compare USB behavior, calibration,
geometry, exposure, timing, and image data — not to introduce differences from the
capture setup itself.

## `NOTES.md` requirements

Each session folder has a `NOTES.md`. Record at minimum:

- capture filename, date/time
- SilverFast version, Windows build
- USBPcap interface used, scanner USB address if visible
- cold boot vs warm session
- scan settings (dpi, mode, iSRD/HDR)
- film/holder position
- exact action performed
- whether the scan completed normally and the carriage returned home
- capture file size, frame count
- any unusual event

Do not write protocol conclusions into the notes unless clearly labeled as
observations for later analysis — keep raw observation separate from interpretation.

## Data integrity checks

Before treating a capture as usable, inspect it with `tshark`/Wireshark and confirm:

- target VID/PID traffic is present
- frame count and duration are reasonable for the action performed
- descriptor traffic, control transfers, bulk-IN/bulk-OUT transfers are present
- large data transfers look intact (not truncated after a few chunks — some
  truncation of bulk-IN payloads by USBPcap on very large transfers is expected and
  not itself a failure, as long as the protocol structure/headers are intact)
- the intended action is actually the one that was captured

A small file is not necessarily invalid, and an expected-size file is not necessarily
valid — check contents, not just size.

## Comparison dataset

The primary comparison dataset is [8200i-se/](../8200i-se/) in this same repo — same
GL128 ASIC, same `bcdDevice` (`0x0702`), different USB PID (`0x1825` vs `0x1824`). Use
it to compare protocol structure, initialization, calibration, motor configuration,
geometry, exposure, and image transfer. A shared ASIC does not mean a register value,
motor table, exposure table, or geometry parameter is interchangeable between models —
every 8100 V2 value must come from an 8100 V2 capture.
