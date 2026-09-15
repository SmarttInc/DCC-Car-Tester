# DC-Car Handheld Tester

Firmware releases for the DC-Car handheld IR analyzer - an ESP32-P4 based
handheld (Waveshare ESP32-P4-WIFI6-Touch-LCD-5, 5" 720x1280 portrait touch)
that captures DC-Car IR traffic live, decodes and names the codes on screen,
stores them in flash, replays them from an on-screen remote pad, and serves a
browser-based code editor over WiFi.

<!-- RELEASE:BEGIN -->
## Latest firmware: v0.5.9 (2026-09-15)

- FIXES A CRASH-WHILE-IDLE, and the "could not start the update task" /
  "could not connect to the update server" failures with it. All three
  were the same illness. Field logs 2026-09-15, fresh boot:
    heap-probe[ctor-pass]: free=52364 largest=31744     (healthy)
    ota: connect failed ... (internal heap 9459 free, largest block 3072)
    H_SDIO_DRV: RX buffer alloc failed (len=3072); dropping read
    rpc_core: Timeout waiting for Resp for [0x101](Req_GetMACAddress)
    Guru Meditation Error: Core 0 panic'ed (Load access fault)
    MEPC: rpc_wifi_get_ps at rpc_wrap.c:1708   MTVAL: 0x00000010
  Internal RAM is fine at boot but collapses to ~9 KB free / 3 KB largest
  once WiFi and the web editor are up. The SDIO driver then cannot get
  the 3 KB buffer it needs, drops reads, and RPC responses are lost - at
  which point esp_hosted 2.12.12 walks a NULL response struct (a load
  from address 0x10) and panics. The network was never broken; it was
  starved. Three fixes, attacking it at every level:
- NO MORE esp_wifi_get_ps(). The power-save governor confirmed each
  transition with a read-back, and that read-back IS the crash site -
  hosted's get_ps wrapper does not null-check its response. The
  component's version is pinned to the C6 firmware and cannot be
  patched, so the defence is to not make the call: set_ps's own return
  code is the answer now. (Same family as the hosted_memcpy(src=NULL)
  panic fixed in 0.5.0 - a different wrapper with the same hole.)
- The MAC address is asked for ONCE and cached. A MAC cannot change,
  but the About card rebuilt this line every 2 s while Settings was
  open - firing Req_GetMACAddress at the C6 forever, on the one page
  where CHECK ONLINE lives. The log shows the cost: a timeout every 5 s,
  each stalling its caller for the full 5 s, all of it congesting a link
  that was already dropping reads. RSSI polling is likewise backed off
  10 s -> 30 s and skipped while the screen is off.
- esp_hosted's task stacks move to PSRAM
  (CONFIG_ESP_HOSTED_DFLT_TASK_FROM_SPIRAM), freeing the internal RAM
  the SDIO buffers are allocated from - hosted runs several tasks at
  5120 bytes each. This is a DIFFERENT knob from the transport mempool,
  which stays internal and must: the SDMMC controller can only DMA out
  of internal RAM (see the note in sdkconfig.defaults). A task stack is
  only ever touched by the CPU, and .text already runs XIP from PSRAM on
  this board, so PSRAM is live whenever code is.
- OTA workers now retry once after yielding, and say why when they still
  fail. A finished OTA task self-deletes, and a self-deleted task's stack
  is reclaimed by the IDLE task, not the scheduler - tapping the button
  again can beat the reaper to it, so a brief yield recovers memory that
  was already free in all but name. If it still fails the message now
  carries the numbers ("needs 10240 bytes, largest free block is 3072"),
  which separates "out of memory" from "fragmented" without a serial
  cable.

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
