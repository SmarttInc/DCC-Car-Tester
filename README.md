# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.2.66 (2026-08-26)

- CHECK ONLINE no longer reports "nothing published" when it never reached
  GitHub at all. A TLS/TCP connect failure now says so ("could not connect
  to the update server") and logs how much internal RAM was free at that
  moment - the field failure it hides was mbedtls_ssl_setup -0x008D, which
  is PSA "insufficient memory" at TLS session setup, not an empty repo.
- The root cause is fixed in the build config: mbedTLS now allocates from
  PSRAM (CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC). A TLS session needs ~40-50 KB
  during the handshake; from internal RAM it competed with the C6 radio
  link's receive pool for the last few KB. This also permanently retires
  the "RX buffer alloc failed at full download speed" class - TLS no longer
  takes part in that fight. Requires a full rebuild to take effect.

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
