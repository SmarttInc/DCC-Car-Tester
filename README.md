# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.3.2 (2026-09-08)

Changes since v0.2.86 :

## 0.3.2 - 2026-09-08
- PERF: receiving from a CONTINUOUS emitter (a function module, the rear
  beacon, the speed-beacon Arduino) made the UI sluggish and delayed the
  on-screen result by seconds. Cause: every ~79 ms burst rebuilt the whole
  Read table (~370 cell writes at ~6.6 Hz on the software renderer).
  Air-driven repeat updates (count/agree ticks) are now coalesced to ~2
  rebuilds/s; structural changes (new code, edits, selection, imports)
  still rebuild immediately, and the big result card still updates on
  EVERY burst - so a code shows up within a tick or two of clean optical
  alignment, and the screen stays responsive while codes stream in.

## 0.3.1 - 2026-09-08
- About gains a "Last rst" line naming why the current boot happened
  (BROWNOUT / POWERON / SW / watchdog...). Purpose: the single-USB-unplug
  restart can now be confirmed as a brownout from the screen alone - no
  cable, no monitor. (Policy refinement: added informational text counts
  as PATCH; MINOR stays for layout/control changes.)
- Diagnosis recorded: the unplug restart is BY DESIGN on this board - the
  battery boost converter (SCT12A0) is held disabled by Q5 whenever USB 5V
  is present, and only starts - through a soft-start ramp - after the rail
  has already collapsed. Firmware cannot bridge it; bulk capacitance
  cannot either (the boost waits for the rail to DIE before starting).
  See the reply / notes for the two real options.

## 0.3.0 - 2026-09-04
- VERSIONING POLICY, adopted from here forward: MAJOR.MINOR.PATCH where
  MINOR bumps whenever something the user SEES or TOUCHES changes (layout,
  pages, controls, displayed values) and PATCH covers everything internal
  (fixes, protocol work, power, drivers). No retroactive recount - the
  changelog itself is the historical record of what each release touched;
  0.2.92 simply becomes the last of the old numbering. The updater
  compares release tags by equality, so the scheme change cannot confuse
  OTA.
- Car page fills the screen edge to edge: the identity and REAR/BRAKE
  cards share the page 3:2, the composer value boxes absorb the identity
  card's share, and the rear card's growth goes into a much taller
  tappable gauge (easier to hit an exact step) and a 72 px brake button.
  No dead space at the bottom.

## 0.2.92 - 2026-09-04
- Car page reorganized into two cards. IDENTITY card: the Type/#/Battery
  composer at half its former height, with its Last detected line, COPY,
  and SEND VEHICLE BURST all inside the same card. REAR/BRAKE card below:
  the STOP..28 gauge, its OWN "Last detected: step N (00 XX XX)" line
  (written only by real captures - tapping the bar changes the armed step
  but never the detection record), and HOLD TO BRAKE.

## 0.2.91 - 2026-09-04
- REAR / BRAKE strip on the Car composer: a STOP..28 gauge shows the last
  rear brake code received (live enough to watch a ramp if the sensor
  holds steady), the amber readout names the step, and the bar is TAPPABLE
  to pick a step by hand - so the transmit half works even without a
  capture. HOLD TO BRAKE repeats the shown code exactly like a real stop
  emitter for as long as it is pressed (00 5C 5C at STOP), separate from
  SEND VEHICLE BURST. Transmits ride the existing hold machinery with a
  new raw-code path (no database entry needed).

## 0.2.90 - 2026-09-04
- SD FIX (field incident): reinserting a card that would not initialize
  was probed forever; the 41st probe wedged the SHARED SDMMC controller
  and took the C6 radio link down with it (sdio_read watchdog storm).
  The insertion watcher now stops after 3 failed probes ("auto-detect
  off - press CHECK CARD" on the Storage card); CHECK CARD re-arms it.
  A card that fails three probes is misbehaving, not slow.
- Rear-code semantics per field decode: labels are now "Rear: braking @
  step N" / "Rear: brake to STOP" (these transmit the LIVE step during
  braking/re-acceleration, followers brake in sympathy). Linear mapping
  step = 0x5C - code CONFIRMED by example (00 52 52 = step 10). car.csv
  rows renamed to match - recopy to the SD card.

## 0.2.89 - 2026-09-04
- NEW VEHICLE CODES: the rear emitter's speed-status broadcasts
  (00 40 40 .. 00 5C 5C: 0x40 = full speed, 0x5C = stop, step =
  0x5C - code) are now recognized by the identify-only resolver and named
  "Rear: speed step N" / "Rear: STOP (step 0)". Until now these were
  silently dropped as unknown. car.csv gained 29 matching rows (recopy it
  to the SD card if you want them name-editable in the database too).

## 0.2.88 - 2026-09-04
- IR RECEIVER ON DEMAND: capture channels are now disabled whenever nothing
  on screen consumes them, and re-enabled the moment something does. The
  receiver runs on the Receiver page, on the Car composer (its
  last-detected readout), and during a Settings > Advanced capture-unknown
  session - everywhere else (home, Remote/Module transmit, Settings,
  standby) the RMT channels are off and IR edges cost zero CPU. Noise
  storms on an idle page no longer burn cycles at priority 10.
- Driver: ir_rmt_rx_pause()/resume() with a race-tolerant receive loop
  (a receive that loses against a pause backs off instead of panicking).

## 0.2.87 - 2026-09-04
- UPDATE FOCUS: the five OTA worker tasks (check-online, URL install, SD
  install, radio flash) now run at priority 12 - ABOVE the IR receive
  task's 10 - so a download preempts IR work instead of the reverse. On
  top of that, both IR callbacks drop edges at the door while an install
  is running: a noisy bench can deliver thousands of edges a second, and
  none of that decode work now competes with the transfer. (What already
  existed and stays: power-save pinned off, RSSI poller paused, screen
  held awake, web page self-suspends its polling.)

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
