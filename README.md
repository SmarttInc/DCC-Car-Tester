# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.2.65 (2026-08-26)

Changes since v0.2.62 :

## 0.2.65 - 2026-08-26
- Module browser names now match the "Function module wiki-en" PDF word for
  word (categories included - B is "Flashing Blue Light Vehicles", T is
  "Lights and Blue Lights"), so the handheld and the original documentation
  read the same. Obvious wiki typos are fixed, and the handful of rows where
  the wiki contradicts the verified hardware (T7/T8's copy-paste error, W5's
  "not used" that really transmits 00 3C 3C) are named from the code they
  actually send, with the discrepancy noted in function_modules.csv.
- Alias info box: arming a module command whose wire code goes by other
  names opens an amber panel right under it - "00 65 65 ALSO COMES UP AS
  Hazard lights OFF (B3), Direction indicators OFF (G6, K6, L6 +4)" - plus
  an "In your database" line whenever your Remote/Car captures know the same
  code under their own name. Commands with a single name show no box.
- Every remaining byte in the category map is hardware-verified (review of
  2026-08-26: N6 = 00 7B 7B, O6/P6 = 00 38 38, T7/T8 = 00 6E 6E/00 6F 6F);
  all VERIFY flags are gone from function_modules.csv.
- function_modules.csv now also carries the icon column, and
  tools/gen_fn_modules.py regenerates main/fn_modules.h from it - edit the
  CSV, re-run the script, rebuild.

## 0.2.64 - 2026-08-26
- Module browser commands now carry icons (stop, pause, play, left/right
  signals, lights, sensors, anti-collision, lanes ...) for at-a-glance
  reading.
- Lane Control corrected per hardware verification: positions 7/8 are the
  Anti-Collision Off/On commands 00 76 76 / 00 77 77 (the front-sensor wire
  codes), NOT a repeat of positions 5/6 (00 3C 3C / 00 3D 3D). The category
  map now marks every remaining wiki-inferred row with VERIFY in
  function_modules.csv so the hardware check can target ~30 rows instead of
  all 216.

## 0.2.63 - 2026-08-25
- Online downloads now ride out WiFi/internet stalls instead of dying at the
  first 15 s lull (the "download stopped at 430/2154 KB" failure): up to two
  minutes of bounded retries, with proper end-of-stream detection so a
  finished download is never mistaken for a stalled one.
- Full-speed downloads no longer crash the radio link. At ~220 KB/s the
  internal RAM pool feeding the C6's receive buffers was exhausted
  ("RX buffer alloc failed ... dropping read" at 95%, then the connection
  died): the TCP window is trimmed 46080 -> 34560, the TLS read buffer
  8 KB instead of 16, and the download loop no longer fires signal-strength
  RPCs over the very link it is saturating (RSSI is logged once at start).

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
