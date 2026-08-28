# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.2.70 (2026-08-28)

Changes since v0.2.68 :

## 0.2.70 - 2026-08-28
- The IR Receiver is now IDENTIFY-ONLY by default. Every layout signal is
  known - the code database, the baked module map, and the verified
  vehicle broadcast model (number 0x80+N, type 0xC0+T, battery 0xE0+B on
  address 00) - so packets nothing recognizes are counted and ignored
  instead of flooding the table. A floating sensor input (op-amp with no
  sensor attached) used to manufacture endless false-positive rows; now
  the Read screen shows a quiet grey "unknown signals ignored: N" line
  and nothing else. Dropped bursts no longer flash the card either.
- Vehicle broadcasts identify THEMSELVES: a car's announcements appear
  named - "Vehicle #3", "Vehicle Type 14", "Battery status 7" - tagged
  Car, straight from the verified model, no capture session needed.
- Recording NEW signals moved to Settings > Advanced > "Capture unknown
  signals" - session-only like admin mode, always off after a reboot.
  While capture is off, the ignored counter points there.

## 0.2.69 - 2026-08-26
- Updates hold everything awake. The screen-off timer already refused to
  fire mid-update; now the radio's power-save governor is also pinned to
  full speed while an update is in flight - before, a STALLED download on
  a dark screen stamped no network activity, so after 60 s the radio could
  legally doze to MIN_MODEM (6x the round-trip) right when the stall
  needed full speed to recover. And if an install starts while the device
  is in standby (webui upload to a dark screen), the screen now wakes so
  the "DO NOT power off" lockout is actually visible.
- WiFi disconnects now log WHY: the C6's reason code with a name for the
  usual suspects (AP idle-kick, beacon timeout, auth fail ...). A bare
  "Station mode: Disconnected" - like the one 18 s after entering power
  save on 2026-08-26 - was undiagnosable.
- DHCP watchdog escalates sooner: one DHCP restart (12 s), then a fresh
  association at 24 s instead of 36 - field data showed restarts bought
  nothing and only the re-associate moved things. Warnings now include
  RSSI so a weak link is visible while the lease is not coming.
- A re-associate the DHCP watchdog itself requested no longer counts
  toward the 4-strike retry limit (the credentials are proven good - we
  were associated). Before, a router whose DHCP stayed quiet for about
  two minutes would strand the handheld in "join failed" until someone
  reconnected by hand; now it keeps retrying indefinitely, while real
  auth failures still stop after 4 tries.

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
