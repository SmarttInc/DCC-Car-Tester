# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.4.0 (2026-09-08)

Changes since v0.3.2 :

## 0.4.0 - 2026-09-08
- IR RECEIVER redesigned as a reception monitor (field request, with
  mockup). The page no longer builds the saved-code table at all - it
  opens instantly - and shows reception as FRAMES: a headline card
  (code name for a lone code, "Frame: N codes" for a volley, wall time,
  packet count), a CODES IN THIS FRAME list - numbered tag-colored
  badge, resolved name, raw hex, per-frame repeat count, XOR/agreement,
  tag - a RAW FRAME DATA strip in exact arrival order, and RECENT
  FRAMES history (last 8; consecutive identical frames collapse into a
  repeat counter instead of flooding at the 79 ms slot rate). Bursts
  landing within 300 ms group into one frame - an MF5 identity volley
  reads as one frame of Number + Type + Battery; 300 ms of quiet
  closes it. Record/Delete/Export left the page.
- NEW: CODE LIBRARY page (Home > CODE LIBRARY) - the full saved-code
  table plus the tag/name editor and RECORD / Delete / Export, moved
  from the Receiver. The table builds only when the page opens and only
  rebuilds when the database changed, so the old table cost is paid
  there, lazily, not on the Receiver. Live-RX green row and blue
  selected row behave as before; IR capture stays active here so Seen
  counts keep climbing while you name things.
- NEW Advanced switch "Tap-to-edit from the Receiver" (session-only,
  default OFF, like Capture/Admin): when ON, tapping a code row in the
  frame view jumps to that code in the Library editor - the bridge that
  keeps the capture-a-new-code workflow one tap long.
- Back button reworked (field request: too hard to hit): no longer in
  the top bar - now a 96 px floating disc pinned bottom-left on every
  inner page, above the content, hidden while the keyboard is up. Pages
  reserve bottom padding so nothing hides under it.
- Home: DEVICE READY / WiFi strip moved to the TOP (field request);
  launcher cards resized to fit the new CODE LIBRARY row.
- Vehicle identity volleys no longer take over the headline card with
  the 3-column CAR TYPE/# panel - the frame rows carry the same facts;
  the Car composer's "last detected" line still updates as before.
- FIX (field report): screen-timeout standby left the BACKLIGHT lit
  behind a black panel. The BSP's panel-sleep hook stops the LCD
  controller but the LED backlight is a separate PWM rail it never
  touches - standby now forces brightness to 0 explicitly on entry
  (wake already restored the user's setting), reclaiming the single
  biggest battery load while "asleep".

## 0.3.5 - 2026-09-08
(No firmware changes in the sim items below - sim work does not bump the
firmware version.)

- Simulator synced to the current handheld. The scenario now feeds a rear
  brake code into the Receiver session (proving the "Rear: braking @ step
  N" resolver end-to-end in read.png), ticks once per burst like the real
  device so vehicle rows pick up their names, and screenshots the About
  card (about.ppm) via a new ui_cfg_scroll_bottom() sim hook. The sim
  build's About now mirrors the device's Battery and "Last rst" lines
  through the host stubs instead of omitting them.
- NEW sim/build-sim.bat: builds the simulator on Windows (needs a
  MinGW-w64 gcc on PATH; clones LVGL v9.2.2 automatically the first
  time). Default build = the INTERACTIVE simulator: ui-sim.exe opens a
  live 720x1280 window of the real UI (mouse = touch) using LVGL's
  built-in Win32 driver - no SDL2 or any other library to install. The
  window auto-zooms to fit the desktop (override: `ui-sim.exe 100` for
  25..200%), and the floating demo-traffic buttons include a new ~Rear
  that injects a descending brake ramp (24, 20, ... 0/STOP, wrap 28) to
  exercise the Car page's REAR strip by hand. `build-sim.bat headless`
  builds ui-sim-headless.exe, which writes the *.ppm screenshot set and
  exits. Startup works around a v9.2 Win32-driver flaw (framebuffer is
  wired in lazily on a 200 ms timer while LVGL's refresh timer pauses
  itself when it runs too early -> permanently black window): the sim
  forces the framebuffer in at the real panel size before frame one.
- lv_conf.h: LV_USE_OS, LV_USE_WINDOWS and LV_USE_LOG are #ifndef-guarded
  so the Windows build can switch them from the compiler command line;
  app-side LVGL calls in the interactive sim take lv_lock()/lv_unlock()
  (no-ops in the SDL/headless builds).

## 0.3.4 - 2026-09-08
- PRIORITY FIX - seconds-late reception on the Receiver page: the RMT
  capture only delivered a transaction on 6 ms of idle OR a full 512-symbol
  buffer. A noisy front end (edges < 6 ms apart continuously) kept the
  transaction open, so a lone remote frame sat captured-but-undelivered
  until noise filled the buffer - about 5 s. The brake ramp looked instant
  because its strong repeating stream filled the buffer quickly; that was
  the tell. RX now uses PARTIAL RECEIVE (IDF en_partial_rx) with a
  128-symbol chunk buffer: chunks stream to the decoder as they arrive,
  and the true burst end is flagged by is_last at the 6 ms idle - so a
  quick remote press shows within a tick or two, and long MF5 volleys
  simply span several chunks. Worst-case delivery under continuous noise
  is bounded by 128 symbols (~1 s) instead of 512 (~5 s).

## 0.3.3 - 2026-09-08
- Smoother scrolling: while a drag or momentum scroll is in flight, ALL
  periodic label rewrites are deferred (WiFi status line, OTA status, the
  2-second live About rebuild - the worst offender, a large multiline
  label invalidated every time the battery millivolts wiggled). Each
  rewrite stole render time from scroll frames, which read as jerkiness -
  worst on Settings, the widget-densest page. Updates resume one tick
  after the finger lifts and the throw settles.

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
