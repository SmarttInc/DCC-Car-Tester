# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.5.8 (2026-09-15)

Changes since v0.5.0 :

## 0.5.8 - 2026-09-15
- WIRE CODES ARE NOW LIBRARY-ONLY. Raw hex ("00 60 60") is gone from
  every operating page - it told an operator nothing the name doesn't say
  better. Cleared from: the Receiver's frame card and its per-code rows
  and RECENT FRAMES lines; the Transmitter's Armed / TRANSMITTING status,
  its code list rows and the Module command rows (which now read "raw
  replay" / "Z frame" instead of code + tag); the remote pad's assign
  sheet and transmit hint; the Module alias box headline; and the Car
  page's braking status, last-detected line and battery buttons. The
  CODE LIBRARY keeps its Bytes column - that page exists to show them.
  - ONE DELIBERATE EXCEPTION: a code with NO NAME has no other identity,
    so its bytes stand in AS the name rather than as an extra column.
    Blanking those would have left unnamed and undecoded captures as
    identical empty rows on the Receiver - useless exactly when the
    analyzer matters most.
  - The Car page's battery token now reads as its decoded index
    ("Battery 0" / "Battery 7") rather than the E0/E7 wire nibble: the
    good/bad meaning of those values is still unproven, so the number
    stands on its own without asserting a mapping.
- Module category pages now use the FULL page height. Inside an open
  category the floating Back bar is hidden (0.4.2) but the page still
  reserved its 120 px gutter, which pushed HOLD TO TRANSMIT up off the
  bottom edge. That space goes to the command list, which is flex-grow
  and absorbs all of it - roughly one more command row visible, and the
  status line and HOLD button sit at the true bottom.
- Fixed: the brake status line read "braking @ step" with the step
  number missing entirely (the value was never formatted in).

## 0.5.7 - 2026-09-15
- The Car page's "Lane:" readout was STICKY (field report): a plain brake
  signal arriving after a lane-wrapped one left the previous lane on
  screen, so there was no way to tell an unassigned rear emitter from an
  assigned one. 0.5.6 updated the readout only when a lane marker
  decoded - nothing ever cleared it.
  Lane assignment is now read as a property of THE BURST, exactly as the
  signal defines it: the whole burst is walked first, then the brake step
  and its lane are published together. A step sandwiched between lane
  markers shows its lane in green; a brake code arriving alone - or two
  brake codes with no marker - shows "Lane: UNASSIGNED" in grey. Before
  any rear signal at all the readout stays "Lane: -".
- The rear card's "Last detected" line now records the lane too, e.g.
  "Last detected: step 10 (00 52 52) +lane RIGHT", so the log line and
  the gauge can never disagree.
- Sim scenario gained a regression guard for this exact case: a plain
  brake burst fed immediately after a lane-wrapped one, screenshotted
  (transmit_rear_plain.ppm) - it must read UNASSIGNED.

## 0.5.6 - 2026-09-14
- LANE ASSIST support in the Car composer's REAR/BRAKE strip, from four
  scope captures (Right/Middle/Left/Special Lane Assist.csv) decoded and
  XOR-verified: a lane-assigned rear emitter wraps its brake step in a
  3-packet volley - lane marker, rear step, lane marker - with one-hot
  lane codes 00 A1 A1 = Right, A2 = Middle, A4 = Left, A8 = Special
  (e.g. Right @ step 28 = A1, 40, A1).
  - RECEIVED row gains a live "Lane: RIGHT/MIDDLE/LEFT/SPECIAL" readout
    next to the step gauge, so you can see whether the car's rear
    emitter is transmitting with a lane assignment and which one.
  - A LANE ASSIST picker (Off / Right / Middle / Left / Special) above
    the RECEIVED row: with a lane selected, HOLD TO BRAKE transmits the
    same 3-packet volley instead of the bare step code.
  - The identify-only resolver now recognizes the 00 Ax Ax family, so
    lane markers are accepted (not dropped as unknown) and auto-name
    themselves "Lane Assist Right/..." in the Library.

## 0.5.5 - 2026-09-14
- Module browser: the top/bottom fade overlays now show ONLY while there
  is actually content hidden on their side. At the top of the list the
  top fade is gone (it was veiling half of the first category button),
  at the end of the list the bottom fade is gone, and a list short
  enough to fit entirely shows neither. Visibility tracks the scroll
  position live.

## 0.5.4 - 2026-09-14
- FIXES THE CAR-PAGE HARD FREEZE (field report: heavy transmitting on the
  Remote page, TX "stopped", then the Car page locked up completely).
  The Car page's SEND button called the blocking transmit STRAIGHT FROM
  ITS LVGL CALLBACK - the exact thing the transmit engine's own comment
  forbids - and its final wait waits for an EMPTY transmit queue. With
  HOLD TO BRAKE (or any held key) continuously refilling that queue, the
  wait never returned and the whole UI froze until reset. The SEND
  callback now only BUILDS the volley; the transmit task sends it and
  frees the buffers. A second SEND while one is queued reports "busy".
- The transmit path is hardened three ways in ir_rmt.c: a mutex
  serializes transmitters (two contexts used to race the shared symbol
  buffer WHILE the encoder was still streaming from it - corrupted
  frames, and the likely reason TX "just stopped"); the final wait is
  bounded by the volley's actual airtime + 500 ms instead of forever;
  and a refused or stuck transmit resets the RMT channel (disable +
  enable purges a wedged loop transmission), so TX heals itself and
  logs "resetting the transmit channel" instead of staying dead.

## 0.5.3 - 2026-09-14
- FIXES THE FIX: 0.5.2's background free-space scan ran at priority 4 -
  ABOVE LVGL's two software render threads (priority 3). Instead of
  freezing Settings like 0.5.1, the multi-second cluster count now stole
  a render core, which read as a LOWER frame rate across pages (field
  report: second Settings visit slower than the first, Library scroll
  sluggish). The worker now runs at priority 1 - below everything that
  draws - so the count only soaks up idle CPU.
- The count also ran on EVERY Settings open. Free space only changes
  when the card does: the result is cached and repaints instantly; a
  fresh count runs only when marked stale - boot, card insert/remove,
  an import, or the CHECK CARD button. While re-counting, the card
  keeps showing the previous numbers instead of "Checking ...".

## 0.5.2 - 2026-09-14
- SETTINGS NO LONGER CRAWLS ON FIRST OPEN (field report: first visit
  near-unusable - the page "slowly scrolls" - second visit fine). Root
  cause: every Settings open ran the Storage card's free-space query on
  the LVGL thread, and esp_vfs_fat_info() counts EVERY free cluster on
  the card - seconds on a big card, during which touches queued up. The
  second visit was only fast because FATFS caches the count once it has
  been computed. The count now runs in a one-shot background task; the
  card shows "Checking ..." and fills in when the answer lands.
- CODE LIBRARY opens fast (same field report): entering the page was
  rebuilding the whole table TWICE (once on entry, then the stale dirty
  flag triggered a second full rebuild one tick later), and each rebuild
  grew the table row by row - an internal realloc of the whole cell array
  per added row. The table is now pre-sized in one shot, refills only
  when something actually changed since the last fill (a new code, an
  edit, a db import, or Seen counts that moved while a receiving page
  was up), and re-entering an unchanged Library costs nothing.
- Tapping a Library row no longer rebuilds the table just to move the
  highlight - the row repaints in place (this was tap lag).
- Another ~8 KB of internal RAM reclaimed (continuing 0.5.1's work, and
  widening the deep-sleep build's boot margin): the record-commit timing
  string (3.6 KB), the transmit task's frame buffers (1.8 KB) and the
  WiFi scan record buffer (2.9 KB) now live in PSRAM.

## 0.5.1 - 2026-09-14
- FIXES THE 0.5.0 BOOT LOOP (assert at port_common.c:53 - the idle task's
  stack failed to allocate before the scheduler even started). Root cause,
  proven by on-device heap probes: esp_hosted runs its ENTIRE host init
  from a C constructor and eats ~99 KB of internal RAM pre-scheduler
  (127 KB free at core-init -> 28 KB after the constructor pass, largest
  block 26 KB, small allocations already spilling into RTCRAM and TCM).
  0.5.0's deep-sleep support linked ~10 KB of additional static internal
  RAM (PMU/sleep-retention suite), which pushed the leftovers below what
  the idle task needs. Not corruption, not sleep code misbehaving - plain
  internal-RAM exhaustion at the worst possible moment.
- Reclaimed 16 KB of internal RAM to fix it, with margin: the OTA task's
  two 4 KB download buffers (main/ota.c) and the DB parser's two 4 KB CSV
  line buffers (main/db.c) were static internal .bss predating the "no
  big buffers off the internal heap" rule; all four now lazily allocate
  from PSRAM on first use.
- The deep-sleep tail of board_poweroff() is compiled out for this build
  (DCC_POWEROFF_DEEPSLEEP=0): auto power-off still runs the full shutdown
  (WiFi stop, panel sleep, backlight off, C6 held in reset) then parks in
  an idle loop instead of entering deep sleep. Flip the define to 1 after
  the bench run confirms the reclaim covers the sleep suite's footprint.
- Boot log now prints two "heap-probe" lines (core-init and ctor-pass)
  showing internal-heap free/largest before and after the constructor
  pass - the early-warning gauge for this failure class. Harmless to
  leave in; they cost microseconds.
- Build system: after "idf.py fullclean", ninja refused to build with
  "unknown target ...dcc_ir_handheld.bin". The bin/ copy step depended on
  the app .bin FILE, which has no named producer rule on a clean tree; it
  now depends on the gen_project_binary TARGET (latent since the copy
  step was added - incremental trees masked it).

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
