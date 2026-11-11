# ESP-IDF ports (ESP32-C6, ESP32, ESP32-S3, CrowPanel 7.0, Arduino Nano ESP32)

Six ESP-IDF 5.3.1 projects share one application. The shared code is in
`ports/esp_idf`: the application in `app/` (start-up, Telnet, network,
OTA, RS-485, the shell bridge, the UART link `link_uart.c` and the USB
Serial/JTAG link `link_usj.c`) and the TinyDesk core as an ESP-IDF
component in `components/tinydesk/`. Each board project picks its link and
holds only what differs: `ports/esp32c6` the USB link, `ports/esp32`
(ESP32 modules with PSRAM), `ports/esp32-4mb` (4 MB boards without PSRAM)
and `ports/esp32s3` (ESP32-S3 with 16 MB flash and Octal PSRAM) the UART
link, `ports/crowpanel7` its LCD, touch and UART0 link
([below](#crowpanel-70)), `ports/nanoesp32` the USB link on an ESP32-S3
([below](#arduino-nano-esp32)), and each its partitions,
`sdkconfig.defaults` and `board.example.conf`. TinyDesk Shell runs in its own FreeRTOS task
and appears as the Terminal app; the desktop runs in the `tinydesk` task.

## Board differences

| | ESP32-C6 (`ports/esp32c6`) | ESP32 with PSRAM (`ports/esp32`) | ESP32, 4 MB (`ports/esp32-4mb`) | ESP32-S3 (`ports/esp32s3`) |
| --- | --- | --- | --- | --- |
| Tested on | ESP32-C6-DevKitC-1 (8 MB) | ESP32-WROVER-IE N16R8 | ESP32-WROOM-32 dev board (ESP32-D0WDQ6, 4 MB) | ESP32-S3 dev board (ESP32-S3R8, 16 MB flash, CH343 USB-UART) |
| Desktop link | built-in USB Serial/JTAG (`ports/esp_idf/app/link_usj.c`, 8 KB RX buffer) | UART0, 921600 8N1, via the USB-UART chip (`ports/esp_idf/app/link_uart.c`, 8 KB RX buffer) | the same `link_uart.c` | the same `link_uart.c` (UART0 on GPIO43/44, the board's UART port) |
| CPU | 1 core, 160 MHz | 2 cores, 240 MHz | 2 cores, 240 MHz | 2 cores, 240 MHz |
| RAM | 452 KB unified (code in IRAM shares it with the heap) | 180 KB internal data RAM + 4 MB PSRAM (of 8 MB) | 180 KB internal data RAM, no PSRAM | 512 KB internal SRAM + 8 MB Octal PSRAM (all of it mapped) |
| Screen limit | 80x25 | 256x96 (buffers in PSRAM) | 80x25 | 256x96 (buffers in PSRAM) |
| Free RAM, desktop and Terminal open | about 113 KB at boot | about 157 KB internal with Wi-Fi up | about 108 KB before Wi-Fi, 75 KB with Wi-Fi up | about 203 KB internal + 7.2 MB PSRAM before Wi-Fi (About, desktop idle) |
| SSH/SFTP server | not in 0.2.0: `ssh start` refuses (it asks for about 160 KB of free internal RAM); a fix is pending | yes | no: it needs about 160 KB of internal RAM (its 64 KB heap plus headroom) | yes |
| Modbus RTU | up to 2 RS-485 lines from the [board configuration](../guide/board-config.md) (`rs485.N.*`) | the same (use UART1/UART2; UART0 is the desktop) | the same | the same |
| Ethernet | W6100 on SPI when `eth.chip = w6100` | the same; not tested on hardware yet | the same; not tested on hardware yet | the same; not tested on hardware yet |
| Console keys | none (USB Serial/JTAG) | `console.uart`, `console.tx`, `console.rx`, `console.baud` | the same as ESP32 | the same as ESP32 (defaults TX 43, RX 44) |
| USB keyboard and mouse | no (the USB port is the desktop's link) | no (no USB) | no | yes, on the chip's own USB port with `usb.host = 1` (`ports/esp_idf/app/usb_hid.c`; ESP-IDF's USB host library and `espressif/usb_host_hid`; one device, no hubs) |
| Bluetooth keyboard and mouse | no: built without Bluetooth (about 35 KB of internal RAM even while off; this board has no PSRAM) | Classic and LE, with `bt.enable = 1` (`ports/esp_idf/app/bt_hid_bluedroid.c`: Bluedroid in dual mode with ESP-IDF's HID host; reports through `hid_input.c`, shared with USB). The image needs an ESP32 revision 3 chip (WROVER-E/IE). | no: built without Bluetooth (the classic ESP32's controller reserves about 64 KB of internal RAM at link time) | the same; tested with a Logitech Pebble M350 |
| Flash | 8 MB: 2 x 3 MB OTA slots, 1.8 MB `/fs`, 64 KB NVS | 16 MB: 2 x 4 MB OTA slots, 7.8 MB `/fs`, 64 KB NVS | 4 MB: one 3 MB app, 960 KB `/fs` | 16 MB: 2 x 4 MB OTA slots, 7.8 MB `/fs`, 64 KB NVS (as on the ESP32 with PSRAM) |
| Software Update, `ota` | yes | yes | no (no second slot): update by flashing | yes |
| Resets on port open | no | yes, with PuTTY (use the [serial bridge](../guide/terminals.md#esp32-boards-without-resets)) | the same | the UART port has the same auto-reset circuit; not checked with PuTTY yet |
| [Pixel canvas](../api/canvas.md) | internal RAM | PSRAM | internal RAM | PSRAM |

## CrowPanel 7.0

`ports/crowpanel7` is for the Elecrow CrowPanel 7.0 (DIS08070H): an
ESP32-S3-WROOM-1-N4R8 (4 MB flash, 8 MB octal PSRAM) with a 7" 800x480
RGB LCD and a GT911 touch screen. The desktop is drawn on the LCD. Each
release has its image: install it from the
[web installer](https://tinydesk-project.github.io/install/#desktop/crowpanel7)
(CrowPanel 7.0), or with esptool
(`tinydesk-desktop-VERSION-crowpanel7-factory.bin` at `0x0`). From source:

```bash
cd ports/crowpanel7
idf.py build
idf.py -p COMx -b 921600 flash      # close the terminal first
```

The Shell edition has no CrowPanel build: the ESP32-S3 one runs on it,
on UART0 at 115200 baud, with the screen dark.

* **Screen:** `main/display.c` keeps a `td_vterm_t` of the LCD's cells
  and paints the cells that changed into a frame buffer in PSRAM, which
  the RGB peripheral scans out. The default 8x16 font gives 100x30 cells;
  `lcd.font` 20, 24 or 32 picks a larger one (Terminus Font, bold): 80x24,
  66x20 or 50x15 cells. Pictures (Sixel) are painted at the LCD's own
  pixels ([Pictures](../api/core.md#pictures)).
* **Touch** (`main/touch.c`, I2C on GPIO19/20) is the mouse: a finger is
  the left button; held still for 0.7 s it becomes the right button
  (context menus), until the finger lifts.
* **Keyboard:** UART0 (GPIO43/44, the board's USB-C port) at 921600 baud.
  With `lcd.mirror = 1` (the default) UART0 also gets the picture, so a PC
  terminal set to the same cell count shows the desktop too. With
  `console.uart = -1` TinyDesk leaves UART0 to a kiosk app.
* **Flash:** one 2.75 MB app and a 1.19 MB `/fs` (room for a kiosk app package and its unpacked copy)
  (`partitions.csv`): no OTA, so update by installing the new release
  without erasing (the files in `/fs` stay). The firmware names its own
  update feed (`TD_UPDATE_BOARD`, `crowpanel7`), so Software Update can
  never offer it the ESP32-S3 image. Code and constants
  run from PSRAM, so the picture keeps going while the flash is written.
* **Pins:** the LCD (GPIO0, 1, 3-9, 14-16, 21, 39-41, 45-48), backlight
  GPIO2, touch GPIO19/20, PSRAM GPIO33-37, flash GPIO26-32, UART0
  GPIO43/44; the TF slot is MOSI 11, MISO 13, CLK 12, CS 10. GPIO38 is
  free. Keys: [Board configuration](../guide/board-config.md#crowpanel-70).
* **Kiosk:** a maintenance start (press RESET, let go, then hold BOOT for
  a moment within 1.5 s) starts without the kiosk app and turns UART0 on
  even with `console.uart = -1`, this time only. A running kiosk app is
  left with a 5-second touch on the top-right corner and root's password
  ([Kiosk mode](../guide/using.md#kiosk-mode)). Sign in still comes first.
* There is no USB keyboard and mouse: GPIO19/20 are the touch screen's.

`tools/sim_lcd.c` (the `tdsim_lcd` simulator) draws the same LCD code on
the PC; see [Tools](../tools.md#tdsim).

## Arduino Nano ESP32

`ports/nanoesp32` is for the Arduino Nano ESP32: an ESP32-S3 (u-blox
NORA-W106) with 16 MB flash and 8 MB Octal PSRAM, the chip of
`ports/esp32s3`. The board has no USB-UART chip: its USB-C connector is
the ESP32-S3's own USB Serial/JTAG port (GPIO19/20), so the desktop is
served there, with the ESP32-C6's link (`ports/esp_idf/app/link_usj.c`),
and UART0 (D0/D1) is free. A community port: contributed and tested on
the board by [aquasock](https://github.com/aquasock) (desktop and window
manager, files, the Editor, `py` with `machine` and `tdui`, a 13-minute
run without dropped frames).

```bash
cd ports/nanoesp32
idf.py set-target esp32s3
idf.py build
idf.py -p COMx flash      # no baud: the link is USB
```

Or install `tinydesk-desktop-VERSION-nanoesp32-factory.bin` from the
[web installer](https://tinydesk-project.github.io/install/#desktop/nanoesp32).

* **The Arduino partition table goes.** The board ships with two 3 MB apps
  and a 9 MB spiffs volume from `0x610000`; this firmware's layout (the
  ESP32-S3's: two 4 MB OTA slots, 7.8 MB `/fs`) covers it, and flashing an
  Arduino sketch back does not restore it. Read the flash out first if it
  holds anything:
  `esptool.py --port PORT --before usb_reset read_flash 0 0x1000000 backup.bin`.
* **Resets go through USB.** `--after hard_reset` toggles RTS, which this
  port ignores, and `--before no_reset` leaves the chip in the loader; if a
  flashed board stays silent, reset it with
  `esptool.py --port PORT --before usb_reset --after watchdog_reset read_mac`
  or unplug it. Plain `idf.py flash` needs neither.
* **Pins:** `TD_LINK_USJ` (set for the whole project) tells Python's
  `machine` that GPIO19/20 are the desktop's link and that UART0 and
  GPIO43/44 are free. The board names its pins D0-D13 and A0-A7; the
  board configuration uses GPIO numbers (`board.example.conf` gives both).
* **Software Update** reads its own feed (`TD_UPDATE_BOARD`, `nanoesp32`),
  never the ESP32-S3 one.
* No USB keyboard (the USB port is the desktop's) and no Shell edition
  build (the shell would need a USB-serial adapter on D0/D1).

## Start-up (`app/main.c`)

1. `quiet_usb_port()`: detach the ROM's `printf` from USB and UART0, turn
   off the ROM banner for warm resets, raise the log level from WARN to INFO.
2. NVS, then `esp_netif_init()` and the default event loop. They come
   first because the Telnet listener, MQTT and Modbus TCP open sockets
   even when no network interface has been started yet; without them a
   board with no Ethernet would crash at boot.
3. The log hook (every `esp_log` line goes to the Log Viewer, nothing
   to the console: `CONFIG_ESP_CONSOLE_NONE`), `fill_sysinfo()`.
4. `tdsh_espidf_init()` (LittleFS on `/fs`, the [board
   configuration](../guide/board-config.md) with the built-in
   `board_builtin.conf`, users, networking), then the
   extra shell commands: `mqtt`, `modbus` (`td_proto_register_shell_commands`),
   Ctrl+C for repeated reads (`td_proto_set_break_check`), `py`
   (`td_py_register_shell_command`, with the same Ctrl+C check through
   `td_py_set_break_check`), `ota`.
5. `td_mb_set_serial(board_rtu_lines())`.
6. Kiosk mode: with `kiosk.app` set, `board_maintenance_start()` (up to
   1.5 s), then `kiosk_app` in `td_sysinfo_t` unless it was a maintenance
   start.
7. On the ESP32-S3, `usb_hid_start()` when `usb.host = 1`: the USB host
   and HID tasks, and `mouse` / `keyboard` in `td_sysinfo_t`. Their keys
   and mouse reports go into a small ring that `link_read()` reads after the
   UART.
8. The `tinydesk` task: `hal_mux_init()`, `td_init()`, apps, Terminal
   backend, `td_step()` loop (priority above the shell).
9. A 30 s timer confirms a freshly installed firmware (OTA trial), and
   `telnet_start()` opens port 23.

## Shared components

### Board interface: `link.h`

```c
bool link_init(void);
int link_read(uint8_t *buf, int cap);        /* bytes received, 0 if none; never blocks */
int link_write(const uint8_t *buf, int len);  /* bytes accepted */
const char *board_platform(void);             /* "ESP32-S3 (UART0, 921600 baud)" */
const char *board_hostname(void);             /* the shell's host name */
const td_mb_serial_t *board_rtu_lines(void);  /* Modbus RTU lines, or NULL */
bool board_maintenance_start(void);           /* start without kiosk mode this time */
```

`board_maintenance_start()` is optional: `main.c` has a weak default that
watches the BOOT button (board key `kiosk.maint_pin`; GPIO0, GPIO9 on the
ESP32-C6) for 1.5 s and is true when it was held for 0.2 s. A port defines
its own when it needs something else. It is only called when `kiosk.app`
is set.

A new ESP32-family board needs only these functions (or one of the two
existing links: `link_usj.c` for a chip's built-in USB Serial/JTAG port,
as on the ESP32-C6, `link_uart.c` for a UART), plus its `sdkconfig.defaults`, partitions and a project
whose `main/CMakeLists.txt` includes `ports/esp_idf/app/app.cmake`; see
`ports/esp32-4mb` for the smallest example.

### HAL and Telnet mux: `hal_mux.h`, `telnet.h`

`hal_mux_init()` returns the TinyDesk HAL on top of the link. While a
Telnet client is logged in, reads and writes go to the client instead, the
local link shows a notice, and the desktop switches to the user who logged
in (and back afterwards).

```c
const td_hal_t *hal_mux_init(void);

void telnet_start(void);            /* read the on/off setting from NVS (default off) */
void telnet_poll(void);             /* accept, log in, notice when the client leaves; non-blocking */
bool telnet_active(void);           /* a logged-in client owns the desktop */
int telnet_read_byte(void);         /* -1 when nothing is waiting */
int telnet_write(const uint8_t *buf, int len);
bool telnet_enabled(void);
void telnet_set_enabled(bool on);   /* saved in NVS */
const char *telnet_peer(void);      /* client address while active */
const char *telnet_user(void);      /* account it logged in with */
```

Everything runs in the UI task: `telnet_poll()` is called on the first
`read_byte` of each pass through the loop.

### TinyDesk Shell in the Terminal: `tdsh_bridge_esp.h`

The shell task's stdin/stdout/stderr are a `funopen()` stream over two
FreeRTOS stream buffers (256 B in, 2 KB out); the Terminal app writes keys
into one and feeds the other through the [terminal emulator](../api/vterm.md).

```c
const td_term_backend_t *tdsh_bridge_esp_backend(void);
bool tdsh_bridge_break_requested(void);   /* Ctrl+C typed since the last call (local console only) */
```

`tdsh_bridge_break_requested()` takes pending input from the stream,
keeps everything except Ctrl+C for the shell, and answers only for commands
running on the desktop's console (not SSH sessions).

### Network, OTA, RS-485

```c
const td_net_ops_t *net_esp_ops(void);          /* Network app: TinyDesk Shell's Wi-Fi/Ethernet, SSH/FTP servers */
const td_ota_ops_t *ota_esp_ops(void);          /* Software Update app */
void ota_esp_boot_ok(void);                     /* confirm the running firmware */
int ota_esp_register_command(void);             /* the root-only `ota` command */
const td_mb_serial_t *rs485_serial(void);       /* the configured RS-485 lines (rs485.N.*), NULL if none */
```

Scans and connects in `net_esp.c` run in short-lived worker tasks; SSH/FTP
server start and stop run on the caller's task so no extra stack eats the
RAM SSH needs. OTA writes the inactive slot, verifies it, and boots it on
trial: it confirms itself after 30 s or on a deliberate restart after 5 s,
otherwise the bootloader rolls back.

## Memory

The ESP32-C6 is the tight one: TinyDesk Shell's SSH server refuses to start unless
96 KB of internal RAM is free (it keeps a 64 KB private heap for wolfSSL).
Rules that keep it there:

* apps allocate their state when their window opens and free it on close;
* the widget pool is `TD_MAX_WIDGETS` (96, about 14 KB) for all windows together, so
  apps draw static labels in `on_draw` instead of using label widgets;
* `CONFIG_FREERTOS_PLACE_FUNCTIONS_INTO_FLASH=y` moves FreeRTOS kernel code
  out of the shared RAM (about 14 KB back);
* protocol buffers exist only while connected.

After boot the C6 has enough free internal RAM for the SSH server (it needs
98,304 B); `heap` shows how much is free.

On the classic ESP32 with PSRAM the 180 KB data segment is too small for
the large screen buffers, so `ports/esp32/main/extram.lf` places the `.bss`
of TinyDesk and TinyDesk Shell in PSRAM (screen, terminal, shell session),
TinyDesk Shell's SSH heap is taken from PSRAM when SSH first starts (its
start check then wants 64 KB of internal RAM), and malloc'd blocks over
4 KB go to PSRAM. About 157 KB of internal RAM is free with Wi-Fi up.

The ESP32-S3 port works like the ESP32 with PSRAM: its
`ports/esp32s3/main/extram.lf` places the same `.bss` in PSRAM, and blocks
over 4 KB go there too. With 8 MB of Octal PSRAM, all mapped, and more
internal RAM, about 203 KB of internal RAM is free at boot.

On boards with PSRAM `main.c` also sets the [pixel canvas](../api/canvas.md)
allocator to PSRAM (`td_canvas_set_allocator()`), so canvases never take
internal RAM; each allocation is logged (`canvas: 15680 bytes in PSRAM`).

The 4 MB port has no PSRAM: everything stays in internal RAM, with the
80x25 screen limit and fewer Wi-Fi buffers (`sdkconfig.defaults`). Its
static data takes 131 KB of the 180 KB segment. The SSH heap is not
static (it is allocated on the first `ssh start`), which is what makes the
port fit; SSH itself does not start there for lack of RAM.

## Configuration that matters

| Option | Why |
| --- | --- |
| `CONFIG_ESP_CONSOLE_NONE` | UART0 may be an RS-485 line (C6) or is the desktop (ESP32, ESP32-S3); logs go to the Log Viewer |
| `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` | OTA trial and rollback |
| `CONFIG_ESP_HTTPS_OTA_ALLOW_HTTP` | updates from a local HTTP server |
| `CONFIG_MBEDTLS_DYNAMIC_BUFFER` | TLS buffers only as large as needed (MQTT over TLS in about 17 KB) |
| `CONFIG_FREERTOS_USE_TRACE_FACILITY`, `CONFIG_FREERTOS_GENERATE_RUN_TIME_STATS` | the Task Manager's task list and CPU % |
| `CONFIG_FREERTOS_PLACE_FUNCTIONS_INTO_FLASH` | C6: RAM for SSH; ESP32: IRAM for the drivers |
| `CONFIG_SPIRAM*`, `CONFIG_SPIRAM_ALLOW_BSS_SEG_EXTERNAL_MEMORY` (ESP32 with PSRAM, ESP32-S3; `CONFIG_SPIRAM_MODE_OCT` on the S3) | PSRAM for malloc and static buffers |
| `CONFIG_ESP_WIFI_*_BUFFER_NUM` (4 MB ESP32) | fewer Wi-Fi buffers in internal RAM |

The TinyDesk component (`ports/esp_idf/components/tinydesk/CMakeLists.txt`)
chooses the screen and widget limits from `CONFIG_SPIRAM`. Versions come
from `PROJECT_VER` in each project's `CMakeLists.txt` (or the `TD_VERSION`
environment variable), shown by About and Software Update.

## Building and flashing

See [Getting started](../guide/getting-started.md). Each project downloads
TinyDesk Shell's pinned third-party components into its own
`managed_components` on its first build.
Partition tables: [Configuration files](../shell/config-files.md).
