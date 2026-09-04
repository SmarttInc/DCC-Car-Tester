# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.2.86 (2026-09-04)

Changes since v0.2.81 :

## 0.2.86 - 2026-09-03
- Fix the 0.2.85 build failure: board_standby() was placed above the WiFi
  state variables it manipulates (s_wifi_suspended and friends live ~300
  lines lower in board.c). Moved below board_wifi_status(), with a stub in
  the no-WiFi build branch and an explicit driver/gpio.h include for the
  BOOT-key read. No behavior change from the 0.2.85 notes.

## 0.2.85 - 2026-09-03
- DEEP STANDBY: when the screen times out (or DISPLAY OFF is tapped) the
  radio now STOPS - WiFi and the web UI are off while the screen is dark -
  and the firmware calls the BSP's panel-sleep hook if this BSP ships one
  (weak-linked; the boot log prints "panel sleep hook found"/"no hook" the
  first time standby engages). Waking restores the panel, brightness, and
  reconnects WiFi with the existing DHCP watchdog. Measured 260 mA standby
  should drop by the radio's share (~30-50 mA) plus the panel's
  (~40-70 mA) where the hook exists.
- Wake paths: tap the screen as before, or press the side BOOT key - the
  key is a guaranteed wake even if a panel hook were to disable touch.
- Trade accepted per field decision: the web UI is unreachable while the
  screen is dark; reconnect on wake takes a few seconds (watchdog covers
  slow DHCP). Update checks/pushes only happen awake.

## 0.2.84 - 2026-09-03
- IDLE DIM: 30 s without a touch drops the backlight to 25% (the user's
  brightness setting is untouched and comes back on the next tap); the
  existing screen-off timeout still follows. The backlight is the single
  biggest battery load - roughly half of the measured ~520 mA idle draw -
  so this claws back a large share of bench-idle battery time. Skipped
  during updates and when brightness is already at/below 25%.

## 0.2.83 - 2026-09-03
- Docs: the charge-STAT mod wire can land on any free J3 GPIO - GPIO48
  (header pin 39) is the physically shortest run from TP1 and is what the
  board.h note now suggests; set PIN_CHG_STAT to the pin actually wired.

## 0.2.82 - 2026-09-03
- Charge-status plumbing: the ETA6098 charger's STAT line (low = charging,
  hi-Z = done) reaches only test pad TP1 on the stock board - no LED, no
  GPIO. After soldering a wire TP1 -> J3 header pin 11 (GPIO5), set
  PIN_CHG_STAT to 5 in board.h and the About battery line gains a live
  "charging" / "charge done" tag. Left at -1 (unwired) the UI shows
  nothing rather than a guess.
- FYI documented in board.h: the stock charge current is ~0.2A
  (ISET R97 = 820k; datasheet points 82k=2A, 150k=1.2A) - about 50 hours
  for a 10 Ah pack. Piggyback 180k across R97 (=147k, ~1.2A) for an
  overnight charge; the MX1.25 battery connector's ~1A rating is the
  practical ceiling, not the cell.

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
