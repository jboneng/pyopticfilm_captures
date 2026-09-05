# Session notes — 04_color_7200

- Capture file: `04_color_7200.pcapng`, 919,800,308 bytes (~877 MiB), 18,788
  frames, 175.570s
- USBPcap interface used: same confirmed interface as sessions 01–03
- USBPcap buffer size raised to 128 MiB before this capture (per large
  bulk-IN volume expected)
- Integrity: **Clean.** All frames are target-device (`07b3:1824`) traffic,
  no unrelated USB noise, no timing gaps >1.7s anywhere in the capture.
- Content:
  - 3,460 control transfers (register read/write + status polling)
  - 15,328 bulk transfers, 918,650,684 bytes total payload — closely matches
    the ~902 MB full-frame 7200 dpi image size expected for this device/mode
  - Bulk size distribution: mostly 0-byte (IRP header only, 7,664 — expected
    on Windows for some chunks) and 124,416-byte payload chunks (7,253 of
    them — the bulk of the image data), plus small setup transfers (512,
    3072, 61952, 125952 bytes) consistent with calibration/shading passes
  - Tail of capture: steady stream of 124,416-byte bulk-IN chunks arriving
    ~22ms apart, continuously, through the last frame at t=172.85s —
    confirms this is a real paced hardware transfer, not a burst artifact
- **Timing observation (not a defect)**: total capture duration is 175.57s,
  vs. ~22.7s recorded for an earlier, non-repo pilot capture of the same
  session type. The earlier pilot capture is documented (see prior
  `INTEGRITY.md` analysis) as having truncated/dropped bulk payloads under
  load — a classic symptom of USBPcap kernel buffer overflow, where delayed
  IRPs get flushed in a burst once buffer space frees, compressing the
  apparent timeline. This capture used a 128 MiB buffer specifically to
  avoid that, and the steady ~22ms-spaced tail above supports that 175.57s
  is the accurate real-world scan duration, not an artifact. Treat this as
  the trustworthy timing reference going forward; flag for follow-up if a
  future capture disagrees.
- Scan settings: 7200 dpi, Color, iSRD OFF, HDR OFF, full 35mm frame, plain
  output
- Film / holder position: TODO — confirm not moved from this point onward
  through sessions 05/06
- Action performed: one full-frame scan, allowed to finish normally, carriage
  returned home before stopping capture
- Outcome: completed normally, no errors, no mechanical anomalies observed
- Any unusual event: none beyond the timing observation above
- Windows build: TODO — fill in
- Date/time: 2026-09-05, ~22:54 local
