# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.5.24 (2026-10-09)

Changes since v0.5.16 :

## 0.5.24 - 2026-10-09
- Car page gains a SIGNALS row in the identity card: a picker (Off / Left
  / Right / Hazard / Flashers) for the status word SEND VEHICLE BURST
  transmits, and "Seen: ..." showing the status word of the last car
  heard (amber for a lit signal, red for Battery Low, grey for off).
  COPY takes the signal and the battery from the received car.
- Status model corrected from 0.5.23: the third broadcast token is ONE
  word, not a bit field. A lit signal REPLACES E0/E7 on the wire (field
  observation 2026-10-09; no E3/E5/E9 were ever captured), and the
  August E7 arrived with no hazards lit, so E7 is "Battery Low", full
  stop - not "hazard + low". Names: E0 Signals Off / Battery Good, E2
  Left Turn Signal, E4 Right Turn Signal, E6 Hazard Lights, E8 Flashers,
  E7 Battery Low; anything else "Status xx (unmapped)".
- Transmit priority: BAD battery sends E7 whatever the signal picker
  says; otherwise the picked signal, otherwise E0. The low-battery-wins
  rule is an ASSUMPTION - what a real car sends with a signal lit and the
  battery low has not been captured. Run that test and the rule follows.

## 0.5.23 - 2026-10-09
- THE VEHICLE'S THIRD BROADCAST TOKEN IS A STATUS BIT FIELD, NOT A BATTERY
  LEVEL (field-decoded 2026-10-09 from the MF5 under-body emitter, IR Car
  Map.xlsx, every row XOR-clean). The handheld now names it from its bits:
  E2 "Left Turn Signal Currently On", E4 "Right Turn Signal Currently On",
  E6 "Hazard Lights Currently On" (= left + right), E8 "Flashers Currently
  On", combinations joined ("Hazard Lights + Flashers Currently On"), E0
  "Signals Off, Battery Good". Bit 0 (E1) stays marked "Battery Low?" - it
  is the one bit still inferred rather than captured (E7 appeared with the
  sense divider floating, E0 with it fed). Battery percentage and voltage
  changes produced no token change on the bench, consistent with CV26 = 0
  (good/bad only).
- Car page "Last detected" reads the status in words ("Hazard Lights +
  Flashers") instead of "Battery N"; COPY takes GOOD/BAD from bit 0 alone.
- Rows still named "Battery status N" from the old reading - in the live
  table or the saved database - are renamed under the new model the first
  time that token is heard again, and the database is saved.

## 0.5.22 - 2026-10-09
- FIXES "hard time picking up codes - the remote barely, the vehicle not
  at all" (field report). Cause: 0.5.17's pairing-phase choice, tested
  only against scope captures of emitters, never against the handheld's
  OWN receiver stream. That stream (every timing string in module.csv)
  opens with a ~1.8-2.0 ms interval before the first preamble half. Too
  long to be a bit half, it made both candidate phases collapse to the
  same alignment, the straddle counts tied, and the slip landed on the
  wrong half for codes with isolated '1' bits: 61 -> 20, 6A -> 20,
  74 -> 30, 31 -> 10, 3A -> 18, 3B -> 19 - all XOR-clean, so accepted,
  then dropped by identify-only as unknown. Six of the 43 recorded module
  codes; the vehicle's 98/C0/E0 broadcasts and the brake steps follow
  the same bit shapes. The phase is now applied after any leading
  interval longer than a bit half. Replaying the 43 handheld-recorded
  timing strings: 43/43 correct (0.5.17-0.5.21: 37/43; the pre-0.5.17
  decoder on the same strings: 37/43 as well, so this is now strictly
  better than anything shipped, not a return to the old behaviour).
- host-test/test_dcc.c: a real handheld capture with its lead-in is a
  permanent regression case.

## 0.5.21 - 2026-10-01
- Settings > Display gains "Dim after" (Never / 15 s / 30 s / 1 min /
  2 min / 5 min; NVS "dim", default the old 30 s). The idle dim to 25%
  used to be a fixed 30 seconds - too soon while reading a frame or
  aiming at a car with the handheld in hand. "Never" keeps the set
  brightness until the screen timeout, which is unchanged.

## 0.5.20 - 2026-10-01
- Build fix: the "Function N  +x.x ms" row text formatted uint32_t with
  %u, which the RISC-V toolchain treats as an error (uint32_t is long
  there). Casts added; no behaviour change. 0.5.19 did not compile.

## 0.5.19 - 2026-09-30
- A single heard code no longer gets a "Function ... or ... or ..." line.
  One code says nothing about a module - a remote sends the very same
  codes - so the head card stays quiet until two or more outputs of the
  same cycle are heard and the map can actually be fitted.

## 0.5.18 - 2026-09-30
- "Slot" is now "Function" everywhere on the Receiver page - the module's
  output number is what you are wiring, so that is the word.
- FUNCTION NUMBERS ARE RESOLVED AGAINST THE CATEGORY MAP. The IR carries
  no output number and two modules are not synchronized, so a lone
  "Speed step 10" packet cannot say whether it came from Function 2 of a
  category-Q module or Function 5 of a category-M one - no receiver could.
  What the handheld knows is the baked map: every category fixes which
  code sits on which output. The codes heard in a cycle, with their
  relative spacing, are laid over every category at every shift, and where
  they fit the shift turns the relative position into the ABSOLUTE
  function number - output 1 no longer has to carry an emitter. Emitters
  on 2 and 4 of a category-A module read "Functions  . 2 . 4 . . . .
  Cat A/L/T" (three categories agree on the numbers, so the numbers are
  certain even though the letter is not). Rows read "Function 4  +16.2 ms".
- When only ONE code is heard the head card lists every output it can
  be: "Function 2 (Q) or 5 (M,BW) or 7 (C,Y)". Two modules elsewhere on
  the layout both sending that code from different outputs will still be
  told apart only if each is heard together with another of its outputs.
- When several shifts fit, or the codes are not on the module map (lane
  markers), numbers are relative to the first packet heard and marked
  "?" - "Function 2?  (from 1st heard)". Wire output 1 in that case.

## 0.5.17 - 2026-09-30
- FUNCTION MODULE SLOT MAP (field request: three lane emitters on one
  module, one detector - which outputs does it hear?). The Receiver page
  now maps every cycle of a function module onto its eight output slots:
  the head card reads "Slots  1 2 . 4 . . 7! 8" (! = heard but corrupted,
  . = silent) and every code row says which slot it sat in and when
  ("slot 4  +24.1 ms"). Rejected packets stay on the map, so two emitters
  overlapping into one detector show up as a bad slot instead of silently
  feeding the "unknown signals ignored" counter. Slots are numbered from
  the first packet heard in the cycle - wire output 1 when testing, or
  read the numbers as relative. Timing from RigolDS1 (2026-08-24, outputs
  1/2/4/8): ~8.5 ms per slot, ~180 us between adjacent outputs, 109.2 ms
  cycle - and the fourth trace is output 8, not 7.
- Car page: "Lane:" is now a MASK - lane markers heard in one cycle read
  "LEFT + MIDDLE + RIGHT", not the last one decoded. A brake volley still
  reports only the lane wrapped around its own step.
- FIXES "only one signal shows, the rest are unknown". Three decoder
  faults, all provable against the scope captures in ir-analyzer/scope:
  - THE UI ONLY EVER SAW THE LAST BURST. The screen polls the decoder
    every 150 ms and read its single "last burst" record; a module lands
    a burst per slot (~8.5 ms), so of outputs 1, 2, 4, 8 only 8 survived
    to the screen. Completed bursts now queue in a 16-deep ring that the
    UI drains completely on every tick.
  - THE STRETCHED END BIT. Two packets ~50-285 us apart (a module's
    adjacent outputs sit ~180 us apart, a lane volley ~115 us) merge the
    idle into the first packet's end-bit space; that period read as a
    START bit, the next preamble became a junk byte (FF, F7, BF...) and
    the checksum failed - every packet but the LAST of a run was
    rejected. A packet whose bytes already verify now closes at the next
    non-zero byte boundary. RigolDS5/DS6 (six-packet MF5 bursts) had
    three of six packets BAD for this reason since day one.
  - THE PAIRING PHASE. Capture starts on the first IR edge - a MARK, the
    second half of a bit - so pairing intervals from zero straddled every
    1<->0 transition and shrank each run of ones by one: 00 A1 A1 read
    00 00 00, 00 5F 5F read 00 0F 0F, 00 9D 9D read 00 0C 0C. The XOR
    check cannot catch it (both echo bytes break identically and still
    cancel), so the wrong reading was accepted as OK. The phase is now
    chosen structurally - straddled pairs sum a '1' half and a '0' half
    (~172 us, between the clean 116 and 228) and the phase with fewer of
    them is the aligned one - and a junk pair resyncs to whichever offset
    leaves the pairs ahead clean, so one corrupted slot no longer knocks
    the rest of the burst out of phase.
  Replayed against every capture in ir-analyzer/scope: Right/Middle/Left/
  Special Lane Assist decode as A1/A2/A4/A8 around a 00 40 40 step,
  RigolDS1-4 as 00 5F 5F (Stop), DS5 as 9D/C0/E7 x2, DS6 as 83/CE/E0 x2,
  DS7 as 00 64 64, and the four-output DS1 combined into one stream as
  5F / 60 / 62 / 72 - category A outputs 1, 2, 4, 8 - all XOR OK, zero
  rejected. Before this release the same files gave 00 00 00, 0F, 0C and
  a BAD packet in every multi-packet burst.
- Decoder records where each packet sits inside its burst and stamps
  bursts from the host clock (the RMT swallows the idle between them), so
  burst-to-burst spacing is real time, not airtime.
- host-test/test_dcc.c: module-cycle and ring cases added; all pass.

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
