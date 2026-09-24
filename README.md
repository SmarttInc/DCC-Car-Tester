# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.5.16 (2026-09-24)

Changes since v0.5.9 :

## 0.5.16 - 2026-09-21
- AUTO POWER-OFF NOW COUNTS THE WEB PAGE AS ACTIVITY. Until now the
  idle timer looked only at the touch screen, so a handheld sitting on
  the bench serving the browser would still shut WiFi down and enter
  deep sleep on schedule - and the browser saw exactly what the field
  report describes: a first load that sits on "loading...", then "this
  site can't be reached", cured by a reset. The 0.5.14 log proves the
  deep-sleep build is what was running (SPIRAM .text 912 bytes, ctor-pass
  free 71180 vs 81044 on 0.5.10), so the unmonitored outage lines up with
  the power-off timer, not with the caching bugs fixed in 0.5.15. Power
  off now requires BOTH the screen and the network (web page polls, code
  edits, OTA) to have been idle for the configured time.
- The web page stops polling while its tab is hidden and refreshes the
  moment it is shown again, so a browser left open in a background tab
  no longer keeps the handheld awake forever, and a foreground tab no
  longer races the poll against a stale reply.
- No change to what is written to the SD card or to the IR protocol.

## 0.5.15 - 2026-09-21
- FIXES "sits on loading... forever, never populates" (field report). Two
  bugs, both introduced by the caching work in 0.5.12, and both of them
  silent - which is why there was nothing in the log to find:
  - AN EMPTY LIST PLUS A 304 IS NOT "NOTHING CHANGED", IT IS NO DATA.
    0.5.12 returned early on any 304 to avoid clobbering what was on
    screen. On a FIRST load there is nothing on screen to protect, so the
    page kept its empty table - and because the fingerprint had not
    moved, every 5-second poll answered 304 as well. Stuck forever, with
    no error anywhere. When the page holds no rows it now bypasses the
    cache outright, so the request cannot come back bodiless.
  - PARSE FAILURES WERE SWALLOWED. 0.5.12 split the fetch and the .json()
    into separate try blocks and the second one returned with no message,
    so a truncated reply parked the page on "loading..." just as quietly.
    Before that split, a bad body at least said "handheld unreachable".
  Every failure path now names itself in the subtitle ("incomplete reply
  from the handheld - retrying...") and leaves the list empty, which
  keeps the 5-second poll trying instead of giving up. The page heals
  itself the moment the handheld answers properly again.
- Verified by reproducing both failures in headless Chromium: a bodiless
  304 and a truncated JSON body each leave the page RETRYING with a
  visible reason, and the table fills the moment a good reply arrives.
  Under 0.5.14 both cases hung silently and never recovered.

## 0.5.14 - 2026-09-16
- "agree" is GONE from /api/codes, and with it an O(n^2) that should never
  have been there. dcc_agree() re-scans the entire code table looking for
  near-miss captures, so producing it for every code was ~59,000 string
  compares per request at 243 codes - and 0.5.12 quietly DOUBLED that by
  hashing the result into the ETag as well. The column was removed from
  the page in 0.5.13, so every one of those cycles was being spent on a
  number nobody reads. (The handheld's own Library still computes it,
  where it is actually shown.) The ETag does not need it either: agree is
  derived from counts, bytes and checksum state, all of which are already
  hashed, so any change that moves agree moves the fingerprint anyway.
- The page itself is now cacheable. It is ~18 KB, changes only when the
  firmware does, and is tagged with the build - so a reload costs one
  small revalidation instead of 18 KB pushed through the SDIO link and
  the radio. A firmware update still lands on the very next load.
- THE PAGE NOW TELLS YOU HOW LONG IT TOOK, and so does the handheld:
  - Under the title: "243 codes on the handheld Â· 180 ms" - the whole
    round trip as the browser saw it.
  - In the serial log: "/api/codes: 243 codes served in 12 ms" - only the
    handheld's share.
  Together they place the blame. Page slow and handheld fast means the
  time is in the radio or the network, not in the firmware - and the most
  likely candidate there is WiFi modem power save, which is deliberately
  entered when the screen is off and the web editor has been quiet, and
  costs the first request after idle a few hundred ms of beacon latency.
  "It feels slow" is not a bug report; "1840 ms and 12 ms" is.

## 0.5.13 - 2026-09-16
- Web editor sorts by TAG as well as name, bytes, most-seen and capture
  order. Blank fields sort LAST rather than first, so unnamed and
  untagged codes collect at the bottom where they are easy to work
  through instead of being the first thing you see.
- XOR and Agree columns removed: this page is the editor, the handheld's
  Receiver is where reception quality gets read. The table is now
  Name / Tag / Bytes / Seen, and its minimum width drops 560 -> 440 px,
  which buys back a phone-sized screen.
  - A FAILED checksum is still shown, because it is not decoration: the
    database keys on (bytes, xst), so a corrupt capture and a good one
    can carry the SAME bytes and would otherwise appear as two identical
    rows with no way to tell which is which. It now reads as a red,
    struck-through byte string with a tooltip naming the fault - the
    information without the column.
- Re-verified in headless Chromium against the 243-code fixture: header
  is Name/Tag/Bytes/Seen, tag sort orders correctly, the 15 bad-checksum
  rows in the fixture render struck through, console clean.

## 0.5.12 - 2026-09-16
- THE 5-SECOND POLL STOPS RE-SENDING EVERYTHING. /api/codes now carries an
  ETag - a cheap FNV-1a fingerprint of exactly what the JSON contains, so
  it moves if and only if the answer would differ. A matching
  If-None-Match gets a 304 with no body and the browser reuses what it
  has. A parked tab went from pulling ~24 KB every five seconds forever
  (and holding the C6 at full power to do it) to a few hundred bytes of
  headers. Hashing 243 codes costs microseconds; sending 24 KB does not.
  Cache-Control is "no-cache", which means "keep it but revalidate" - it
  is what makes the 304 possible, not an instruction to discard.
- CATEGORIES in the web editor, mirroring the handheld's Library filter,
  because 243 codes in one flat list is not a list, it is a haystack:
  - Filter chips (All / Remote / Car / Module / Untagged) with live
    counts. A chip means "everything carrying this tag", so chips
    OVERLAP - a Remote+Car code counts under both and the chip numbers
    deliberately total more than the list.
  - Group headings when the whole list is shown. A group is the EXACT tag
    combination, so groups partition the list and their counts do add up.
    The group is also the primary sort key, or every other row grows a
    heading of its own.
  - Search across name, bytes and tag; sort by most-seen, name, bytes or
    capture order. Grouping switches off while searching or sorting by
    capture order, where it would only be noise.
  - The subtitle reads "showing 12 of 243 codes" whenever a filter is on.
- Also: unnamed rows show a "name this code" placeholder instead of an
  empty box; Ctrl/Cmd+S saves; and closing the tab with unsaved edits now
  asks first - losing a page of renames is the kind of thing you forgive
  exactly once.
- Fixed while adding the above: the tag popup found its button by DOM
  position, which stopped matching the row index the moment the table
  could be filtered or grouped. It looks up the row by data-k now.
- Verified by rendering the real page in headless Chromium against a
  243-code fixture: chip counts, group partitioning (41+40+41+81+40=243),
  search, filtering and sort all checked, console clean. That test caught
  the grouping bug above, which had produced 470 rows for 243 codes.

## 0.5.11 - 2026-09-16
- WEB EDITOR LOADS FAST (field report: slow, worst on the first try after
  boot). Two causes, both per-request overhead rather than real work:
  - /api/codes sent ONE CHUNK PER CODE - 245 separate chunked-encoding
    frames for this unit's 243 codes, each its own socket write, each its
    own transfer over the SDIO link to the C6 before reaching the air. On
    a first load the TCP connection is new, so all 245 tiny writes crawl
    through slow start, every one small enough to trip the Nagle/
    delayed-ACK stall. The JSON is now built in a PSRAM buffer and
    flushed in ~4 KB pieces: ~245 writes become ~6, identical bytes on
    the wire. If the buffer cannot be allocated it falls back to the old
    path, so a low-memory moment degrades to slow, not to broken.
  - The HTTP server ran with lru_purge_enable off. Its max_open_sockets
    is 7 and THREE of those are reserved for the server's own use, so
    only four browser connections could exist - and a browser opens up to
    six for a single page. Past four, the server REFUSED new connections
    and the browser sat on its own retry timer, which reads exactly as "a
    slow first load". Purging the least-recently-used idle socket makes a
    new connection always succeed immediately. (max_open_sockets stays at
    7 deliberately: CONFIG_LWIP_MAX_SOCKETS is 10 and SNTP plus the OTA
    HTTPS client need the rest.)
- Still outstanding and bigger than either: the page re-fetches the ENTIRE
  code list every 5 seconds while a tab is open. An ETag on /api/codes
  plus 304 handling would cut that to nothing when the list has not
  changed - worth doing, but it changes what the browser caches, so it
  belongs in its own version rather than riding along with a transport
  fix.

## 0.5.10 - 2026-09-15
- Boot log gains a third heap line, "heap-probe[wifi-up]", printed once
  the radio has an address - WiFi up, web editor listening, UI built.
  The two boot probes measure a machine that is not doing anything yet;
  this one measures the state the SDIO buffers, the TLS handshake and
  the OTA task stack actually have to fit inside. It read ~9 KB free /
  3 KB largest when the handheld was panicking in 0.5.6, so it is the
  single number worth watching. No behaviour change.
- For the record, what 0.5.9 measured on hardware (same board, cold
  boot, same 243-code card):
    ctor-pass internal free  52364 -> 81044   (+28 KB reclaimed)
    ctor-pass largest block  31744 -> 40960
    boot to ready           18009 ms -> 1904 ms
    transmit tab build       5740 ms ->  199 ms
    IDLE0 task watchdog      triggered -> gone
  The boot-time collapse was not an intended effect and is worth
  understanding: with SPIRAM_MALLOC_ALWAYSINTERNAL at 16384, every
  allocation under 16 KB is attempted in internal RAM first. Once that
  pool is nearly full, each of the thousands of LVGL allocations walks
  the whole fragmented internal heap, fails, and only then falls back to
  PSRAM. Restoring headroom did not make allocation faster; it stopped
  every allocation from failing first.

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
