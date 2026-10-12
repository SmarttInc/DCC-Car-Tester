# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.5.25 (2026-10-12)

- FIXES the web editor going unreachable and OTA failing with "select()
  timeout" after hours of use (field report: internal heap 9551 free,
  largest block 5376, at 5 h uptime). The BSP example's sdkconfig gave
  every TCP socket a 34560-byte receive window and a 23040-byte send
  buffer, allocated from INTERNAL RAM: a browser tab holding a few
  keep-alive connections could pin ~50 KB of the P4's ~140 KB, and the
  esp_hosted SDIO link then could not get DMA buffers - the same
  starvation class as 0.5.9, reached by a different road. sdkconfig and
  sdkconfig.defaults now set CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y (lwip
  pools from PSRAM first; the SDIO path copies through hosted's own
  internal mempool, so PSRAM payloads are safe) and 11520-byte TCP
  buffers (8 x MSS - a 25 KB code list streams fine). REBUILD FROM A
  CLEAN CONFIGURE: both files changed.
- One heap line per minute on the serial log ("heap: t=.. internal free /
  largest / min | psram | tasks"), so the next report carries a trend
  instead of a snapshot. A steadily falling "min" is a leak; a "free"
  that dips with web traffic and recovers is buffering.

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
