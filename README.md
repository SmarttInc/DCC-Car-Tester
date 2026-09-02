# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.2.81 (2026-09-02)

Changes since v0.2.78 :

## 0.2.81 - 2026-09-01
- IR TX pad now set to maximum drive strength (GPIO_DRIVE_CAP_3, ~40 mA
  class). This is a STOPGAP for emitters wired directly to the pin - the
  real range fix is a one-transistor driver (recipe in board.h): LEDs in
  series from 5 V through ~22 ohm, switched low-side by an NPN or N-MOSFET
  whose gate/base comes from PIN_IR_TX. Same polarity, no firmware change.

## 0.2.80 - 2026-09-01
- Battery calibration applied from a field meter comparison (device 4.05 V
  vs DMM 4.03-4.04 V): BATT_CAL_MV = -15. An offset trim is sufficient - a
  divider gain error of this size would differ from a pure offset by under
  2 mV across the whole 3.45-4.2 V operating window.
- The Settings/About battery line now refreshes every 2 s WHILE the page is
  visible (it used to be a snapshot taken when the page opened), so a
  meter-vs-display check is a live comparison.

## 0.2.79 - 2026-09-01
- Battery gauge now uses a proper equilibrium-OCV curve for a 1S LiPo
  (fuel-gauge characterization shape: 50% = 3.85 V, 30% = 3.77 V,
  20% = 3.74 V) instead of the earlier hand-set floors. Empty is now the
  protective 0% = 3.45 V rested - below that the cell holds ~2% and is
  being damaged. Note: pack CAPACITY does not bend a voltage curve; at
  this device's ~0.04C drain the reading is effectively resting OCV.

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
