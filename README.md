# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.2.62 (2026-08-25)

Changes since v0.2.60 :

## 0.2.62 - 2026-08-25
- Online download: reads sized to the CDN's 16 KB TLS records (heap buffer),
  and the log now prints a time-accounting split at the end - how many ms
  went to network/TLS vs flash writes vs everything else - so download-speed
  questions get answered with numbers instead of theories.

## 0.2.61 - 2026-08-25
- CHECK ONLINE now asks first: it reports "Version x.y.z found" in a centered
  popup with INSTALL / NOT NOW, and downloads nothing until you confirm.
- Any device-firmware install (online, SD, or web upload) locks the handheld
  behind a full-screen "Updating firmware - DO NOT power off" popup with
  progress, and the web editor shows a matching "Update in progress" overlay
  - including installs started from the other side.
- Successful installs RESTART THE HANDHELD AUTOMATICALLY; the manual
  "reboot into new firmware" button is gone from both UIs.
- The Module area is now a category browser, styled after the official
  function-module app: the A-U categories with colored letter badges and
  dipswitch labels, each opening its 8 clearly-named commands - tap to arm,
  HOLD to transmit. Commands already captured use their recorded raw
  timings; the rest are added to the database on first use and transmit as
  synthetic frames. "All captured codes" keeps the flat list one tap away.
- release.ps1: release notes now cover EVERY changelog section since the
  previous GitHub release, so skipped versions tell their full story.

Download `dcc_ir_handheld.bin` from the [latest release](https://github.com/SmarttInc/DCC-Car-Tester/releases/latest), or on the handheld: **Settings > Firmware > CHECK ONLINE**.
<!-- RELEASE:END -->

## Updating a handheld

Three channels, pick whichever is closest to hand:

- **Over the air** - on the handheld: Settings > Firmware > CHECK ONLINE.
  It reads this repository's latest release and installs the `.bin` if it is
  newer than what is running.
- **From a browser** - open the handheld's web page (the IP is shown on the
  Settings tab), click the gear, and upload `dcc_ir_handheld.bin`.
- **From the SD card** - copy `dcc_ir_handheld.bin` onto the card, insert
  it, then Settings > Firmware > FROM SD CARD.

Every channel goes through the same safety path: the image header is
validated before a byte is flashed, the write goes to the inactive firmware
slot, and the boot switch happens only after the whole image verifies. A new
firmware that fails to start is rolled back automatically on the next power
cycle, and Settings > Advanced has a manual ROLL BACK button that boots the
previous firmware at any time - nothing is erased in either direction.

## What is in a release

- `dcc_ir_handheld.bin` - the plain ESP-IDF app image, exactly what every
  update channel expects. No merged binary, no bootloader, no partition
  table.
- The release notes are the matching section of the project's change log.
