# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.2.78 (2026-08-28)

Changes since v0.2.70 :

## 0.2.78 - 2026-08-28
- Car page polish: Type and # values sit in their OWN bordered box, clearly
  separate from the arrow buttons; the arrows grew to 150 px tall (so the
  value box is shorter), and the captions state the valid ranges -
  CAR TYPE (0-15), CAR # (0-31).

## 0.2.77 - 2026-08-28
- The Car page is now ALL composer: the stored-code list and the HOLD TO
  TRANSMIT button are gone from that tag entirely - SEND VEHICLE BURST is
  the one transmitter there, and the panel fills the page.
- Car Type and Car # are big up/down stepper buttons around a large bold
  value (wrap-around at the ends) instead of scroll wheels, and Battery is
  a pair of toggle buttons - the selected token lights up (GOOD/E0 green,
  BAD/E7 red).

## 0.2.76 - 2026-08-28
- The Read card goes BOLD for vehicles: when a broadcast volley lands, the
  big green card splits into three large-type panels - CAR TYPE / CAR # /
  BATTERY - replacing the yellow burst box entirely (normal traffic still
  gets the classic card and banner). Battery shows its raw token (E0/E7)
  until the good/bad mapping is settled.
- Car tab gains a VEHICLE COMPOSER: pick Car Type (0-15), Car # (0-31) and
  Battery GOOD (E0) / BAD (E7), then SEND VEHICLE BURST transmits the same
  6-packet volley a real car broadcasts (number -> type -> battery, twice,
  ~115 us between packets, straight from the RigolDS5/DS6 captures). A
  "Last detected" line mirrors the receiver live, and COPY loads those
  values into the pickers.
- No lookup file needed: identification comes from the verified broadcast
  model. car.csv (all 64 vehicle codes, named) is provided anyway for the
  SD import / irlog / exports, so those carry the same names.

## 0.2.75 - 2026-08-28
- Signal recognition is near-instant with the op-amp front end. The scope
  capture (RigolDS7) showed the comparator skews duty cycle hard: a '1'
  bit arrives as 33 us mark + 81 us space instead of 58/57 - and 81 us sat
  exactly ON the per-half 80 us classification boundary, so most bursts
  decoded as junk, identify-only silently dropped them, and recognition
  "took a second or two" (it was really failing until a lucky repeat).
  Bits are now classified by PERIOD (mark+space: ~115 us = '1', ~230 = '0')
  which no duty-cycle skew can touch; the skewed waveform is pinned by a
  host test, and the old pulse-distance fallback stays as a safety net.
- UI reacts in 150 ms instead of 500: a decoded burst reaches the screen
  on the next tick (change-gated, so idle ticks stay nearly free).

## 0.2.74 - 2026-08-28
- Battery percentage is now a real LiPo gauge. The old linear 3.3-4.2 V
  map burned ten points in the first tenth of a volt (a LiPo holds ~40%
  of its charge between 4.2 and 3.9 V) and wobbled with ADC noise - the
  bench sweep showed 99 -> 95% at a constant 4.20 V. Voltage now maps
  through a 21-point LiPo discharge curve (valid at this pack's tiny
  ~0.05C load), readings are EMA-filtered on top of the 8-sample burst,
  and the displayed value is rate-limited so it moves smoothly.
- Floors per the field request: 100% = 4.20 V, 1% = 3.35 V, 0% = 3.30 V
  (conservative LiPo safety floor). "USB / no battery" is only claimed
  below 2.0 V - a floating divider reads near zero; 2.5-3.3 V used to show
  "USB" when it is actually a nearly-dead battery.
- About now shows the measured pack voltage next to the percentage, and
  BATT_CAL_MV (board.c) trims divider tolerance: compare About against a
  meter once and set the difference.

## 0.2.73 - 2026-08-28
- Every boot now logs WHY it happened (reset reason: POWERON / BROWNOUT /
  SW / PANIC / watchdogs). Field question: pulling either USB cable
  restarts the unit even on battery - BROWNOUT in this line proves the
  rail sags during the USB-to-battery power-path handoff (a board-level
  behavior firmware cannot prevent, but capacitance across the battery
  input can soften); POWERON would mean the supply is simply cut.

## 0.2.72 - 2026-08-28
- The RSSI poll can no longer strangle a struggling link. Signal strength
  is now a CACHED value the WiFi supervisor refreshes at most every 10 s
  (link healthy, no update running); the status bar and every log line
  read the cache. Before, each reader fired a radio RPC - the 2026-08-28
  web-upload wedge showed the status bar's 5.5 s poll timing out in 5 s
  beats against the dying link, hammering it exactly while it tried to
  recover.
- The web editor stops its 5 s background polling while ITS OWN upload is
  running. Every extra open socket carries a full TCP receive window of
  internal RAM, and that pressure is what dropped an SDIO read mid-upload
  ("RX buffer alloc failed (len=9216)") and wedged the transfer at 514 KB.

## 0.2.71 - 2026-08-28
- Online updates now RESUME instead of dying. Field log 2026-08-28: a weak
  link (rssi -51) stalled at 735/2165 KB and GitHub's CDN reset the silent
  connection ~30 s later - no amount of read-patience survives that. A
  broken connection is now reopened with an HTTP Range request for the
  remainder (up to 5 reconnects) and the flash sink keeps writing where it
  stopped; a release replaced mid-download is caught by a size check, and
  a server that ignores Range restarts cleanly from zero. Read-patience
  per connection drops from 2 min to 45 s - past that the connection is
  dead anyway and reconnecting is the only move that works.
- Transient connect failures during an install ("HTTP -1": connection
  opened, server never answered) retry instead of aborting, and the error
  message finally says what happened.

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
