# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.5.27 (2026-10-12)

Changes since v0.5.25 :

## 0.5.27 - 2026-10-12
Full code and config review (four parallel reviewers, every finding
re-verified by hand; host tests, the headless simulator and the embedded
page in headless Chromium all pass). What changed:

DECODER (host-tested, every scope capture and all 43 handheld-recorded
strings still decode)
- A corrupted packet stays BAD. The period-model fallback pairs the
  straddling phase on the real wire, so a single flipped bit used to come
  back as a DIFFERENT, XOR-clean code (12 of 24 single-bit flips of
  00 74 74 decoded as "OK 00 30 30" - a real module code). The fallback
  now runs only when the symmetric scan saw no packet shape at all.
  Regression test added.
- Packet close requires three bytes (addr + cmd + xor): "00 00 20" no
  longer closes early as an OK two-byte "00 00". Short fragments no longer
  consume the last of the eight packet slots, so ten runts ahead of a real
  packet cannot hide it.
- Burst ring 16 -> 64: a module lands a burst every ~8.5 ms and the UI
  drains every 150 ms; at 16 the producer lapped the reader every tick
  (torn records, ~18% of bursts never seen in a host replay). The reader
  now copies each record, keeps four slots of margin, re-checks it was not
  lapped, and pairs the producer's barrier with an acquire load.
- dcc_delete() fixes up every index still in flight (last_code, the
  burst list, the ring) so a web delete cannot make the card name the
  wrong code.
- Exact-fit off-by-one in dcc_build_timings.

MODEL LOCK (new)
- One recursive mutex around the code table and the database, taken by
  the RMT rx task (per interval), the LVGL task (naming, delete, SD
  import, the 0.5.23 migration), the transmit task (timings copied under
  it, never parsed live) and every web handler. Before this, a web save
  memmoved 40 KB of the table while the rx task appended to it, two
  db_save() writers could interleave on one SPIFFS file, and h_csv could
  read a timings string the handheld had just freed.
- A web save now refreshes the handheld's own lists (it used to show the
  old name until the next structural change).

UI
- Firmware updates lock the screen, wake it and restart the handheld on
  EVERY page. That logic lived behind "Settings is visible": a browser
  upload on the Home page installed silently and never rebooted.
- Idle dim no longer restores full backlight on a panel the screen
  timeout just put to sleep (with the default 30 s / 2 min the screen-off
  saving was being thrown away).
- Lane mask resets on every new module cycle, not only when the cycle
  happened to open with a lane marker.
- Brightness slider writes NVS once on release, not per drag event.
- Cycle map slimmed from 2.2 KB of internal .bss to ~400 B; vehicle
  field class cached instead of re-parsed 100+ times per repaint.
- Memory barriers on the Car-volley hand-off and the storage worker's
  text; the transmit task's start is checked; the migration's database
  save happens once after the drain loop, not per packet.
- Before auto power-off: capture paused, any held key released.

DEVICE SIDE (compiler on your machine first)
- SD watcher: its own probe re-armed the watcher every lap, so the
  "auto-detect OFF after 3 failed probes" stop NEVER fired - the exact
  shared-controller wedge it exists to prevent. Fixed.
- WiFi: after four fast retries the join was FAILED for the rest of the
  session (an AP reboot = web editor unreachable with a healthy heap).
  Now re-joins forever with 15 s -> 5 min backoff; wrong-password reasons
  still stop.
- Optional hosted RPCs (RSSI, MAC) turn OFF for the session after the
  first miss; the MAC is asked at most once per boot (it was re-fired
  every 2 s from the LVGL thread when it failed).
- Park: the C6 reset line now follows CONFIG_ESP_HOSTED_SDIO_RESET_
  ACTIVE_HIGH - it was driven to "run", leaving the radio alive on a dead
  link (part of the 70 mA).
- ota_mark_valid() moves to after the first UI tick (RMT, SD watcher and
  WiFi started) instead of before them.
- ota_spawn() is finally wired to all five update tasks (the 0.5.9 retry
  was dead code). Download stalls are recognised (-ESP_ERR_HTTP_EAGAIN),
  so the two-stall patience actually engages.
- RMT receive: chunks are copied in the ISR instead of queued as pointers
  into a buffer the driver reuses (edge corruption under load); stale
  queue entries are flushed after a pause; a lost final chunk re-arms the
  channel after 2 s instead of parking capture forever.
- Timing strings live in PSRAM (strdup put ~450 B per recorded code in
  internal RAM - ~100 KB for a full table). Names and tags are cleaned of
  tabs and line breaks at the door (one tab corrupted the TSV database).
- No internal-RAM fallback for the three big tables: PSRAM or abort.

WEB EDITOR
- JSON escapes control characters (a tab in a name made the page stick on
  "incomplete reply" forever); HTML escaping covers quotes and the bytes
  column (a crafted CSV could inject markup); names capped at 63 chars.
- Reload after an update waits for a DIFFERENT build stamp instead of the
  first answered request - a ~100 ms race used to reload the OLD firmware.
- Save/upload buffers in PSRAM; a failed flash write releases the OTA
  handle; export checks fopen and removes its temp file on failure;
  /api/sysinfo no longer walks the SD card's free clusters every 5 s and
  counts as activity for the power-save governor; 4 server sockets.

CONFIG (sdkconfig + sdkconfig.defaults, full clean)
- TCP MSL 60 s -> 10 s, sockets 10 -> 16, httpd header scratch 1 -> 2 KB,
  purge buffer 32 -> 1024 B, task WDT 5 -> 10 s, hostname dcc-handheld,
  PSRAM boot memtest off, LVGL examples/demos not built.

## 0.5.26 - 2026-10-12
- REVERTS the lwip-pools-in-PSRAM setting from 0.5.25
  (CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP back OFF). That build took a DHCP
  lease and then answered neither ping nor HTTP from the same subnet: the
  esp_wifi_remote / hosted transmit path and PSRAM pbufs do not mix on
  this stack, whatever the reasoning said. The smaller TCP buffers stay
  (11520 each way, down from 34560 / 23040) - they cut a socket's internal
  footprint from ~57 KB to ~23 KB on their own. sdkconfig.defaults now
  carries one setting for them instead of two that disagreed. Full clean
  before building. Firmware code unchanged from 0.5.25.

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
