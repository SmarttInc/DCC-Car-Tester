# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.5.0 (2026-09-11)

Changes since v0.4.0 :

## 0.5.0 - 2026-09-11
- AUTO POWER-OFF (field report: unit found at 2.5 V after days idle - deep
  standby still ran the P4 at full clock). Two triggers, both ending in
  ESP32-P4 DEEP SLEEP - the practical OFF, since the board's ECJ23001
  power latch has no GPIO and true off cannot be commanded from software:
  - Long idle: Settings > Display > "Auto power-off" (Never / 1 h / 4 h /
    12 h, DEFAULT 4 h, NVS-persisted). Counts from the last touch, fires
    from standby, never during a firmware update.
  - Critical battery: below 3.30 V sustained for 60 s, or below 3.10 V at
    all, powers down immediately whatever is on screen (a charger lifts
    the reading and self-clears the trigger).
  Waking up = the RESET side key, or PWR long-press (hard off) then press.
  board_poweroff() runs the full standby path first (WiFi stop, panel
  sleep, backlight 0), then sleeps forever.
- board.h gains PIN_C6_RESET, set to 54 - confirmed from the device's own
  boot log ("Reset slave using GPIO[54]"), no menuconfig digging needed.
  Power-off holds the C6 in reset through deep sleep, removing its ~20 mA
  idle-on-a-dead-link draw. Set to -1 to disable.
- Library page is now PASSIVE (field report: it went sluggish while codes
  streamed in). IR capture runs ONLY on pages that display reception - the
  Receiver, and the Car composer's last-detected readout. Everywhere else
  the RMT channels pause: no decode work, no table rebuilds. The Library
  is just the list; Seen counts advance only while a receiving page is up.
- Standby no longer stops WiFi - the webUI stays reachable while the
  screen sleeps (adaptive power save still drops the radio to MIN_MODEM
  when idle). This also removes a crash path caught in the 2026-09-11
  field log: a timed-out Req_WifiStop during standby entry followed by a
  wake-up resume left esp_hosted's RPC layer confused, and the next status
  poll died on hosted_memcpy(src=NULL) - a hard panic and reboot. WiFi now
  stops exactly once, inside board_poweroff(), where nothing ever resumes
  (belt: the RSSI poll also skips while WiFi is deliberately down).

## 0.4.2 - 2026-09-08
- Back control is now a full-width rounded BAR across the bottom of every
  inner page (third shape's the charm) - same keyboard-aware hiding.
- IR RECEIVER is fully STATIC: the frame card, the 5 code slots and the 6
  history lines are fixed-size widgets updated in place, so nothing on the
  page moves or resizes as different frames come in (also cheaper: zero
  widget churn per burst).
- Module browser: top and bottom of the category/command lists fade out
  slightly, so clipped rows read as "more to scroll" instead of ending
  abruptly at the Back bar.
- Inside an open Module category the main Back bar HIDES: the list's own
  BACK TO CATEGORIES button is the only way out, so you step back through
  the browser instead of accidentally jumping to the launcher.
- Car page REAR/BRAKE split into two bars: a slidable TRANSMIT step picker
  (drag to choose what HOLD TO BRAKE sends; enabled from boot) and a
  display-only green RECEIVED gauge showing what the rear emitter is
  actually broadcasting - setting a value no longer fights the live
  capture overwriting it.

## 0.4.1 - 2026-09-08
- Receiver page refinements from first field use of 0.4.0:
  - The bursts/codes/db status line and the unknown-ignored note are
    PEGGED to the bottom edge of the page (grow spacer; the page is a
    fixed layout now, no scrolling), indented clear of the Back pill.
  - CODES IN THIS FRAME caps at 5 rows; a frame carrying more distinct
    codes says so in the headline ("Frame: 5+ codes") instead of
    silently truncating. RECENT FRAMES history trimmed 8 -> 6 rows.
  - RAW FRAME DATA strip removed - every row already shows its raw hex,
    so the strip repeated information.
- Back control is now a large rounded pill labeled "Back" (same
  bottom-left spot, still keyboard-aware) - the bare 96 px chevron disc
  read as decoration, not a button.

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
