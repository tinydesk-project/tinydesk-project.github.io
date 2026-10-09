# ESP-IDF ports (ESP32-C6, ESP32-S3, ESP32)

Four ESP-IDF 5.3.1 projects share one application. The shared code is in
`ports/esp_idf`: the application in `app/` (start-up, Telnet, network,
OTA, RS-485, the shell bridge, the two desktop links) and the TinyDesk
core as an ESP-IDF component in `components/tinydesk/`. Each board
project holds only what differs: its choice of link (`ports/esp32c6` and
`ports/esp32s3` the built-in USB Serial/JTAG port, `app/link_usj.c`;
`ports/esp32` for ESP32 modules with PSRAM and `ports/esp32-4mb` for 4 MB
boards without PSRAM UART0, `app/link_uart.c`), and its partitions,
`sdkconfig.defaults` and `board.example.conf`. TinyDesk Shell runs in its own FreeRTOS task
and appears as the Terminal app; the desktop runs in the `tinydesk` task.

## Board differences

| | ESP32-C6 (`ports/esp32c6`) | ESP32-S3 with PSRAM (`ports/esp32s3`) | ESP32 with PSRAM (`ports/esp32`) | ESP32, 4 MB (`ports/esp32-4mb`) |
| --- | --- | --- | --- | --- |
| Tested on | ESP32-C6-DevKitC-1 (8 MB) | a board with an ESP32-S3R8 (rev 0.2: 8 MB octal PSRAM in the chip, 16 MB flash) | ESP32-WROVER-IE N16R8 | ESP32-WROOM-32 dev board (ESP32-D0WDQ6, 4 MB) |
| Desktop link | built-in USB Serial/JTAG (`ports/esp_idf/app/link_usj.c`) | the same `link_usj.c` (the board's "USB" connector) | UART0, 921600 8N1, via the USB-UART chip (`ports/esp_idf/app/link_uart.c`, 8 KB RX buffer) | the same `link_uart.c` |
| CPU | 1 core, 160 MHz | 2 cores, 240 MHz | 2 cores, 240 MHz | 2 cores, 240 MHz |
| RAM | 452 KB unified (code in IRAM shares it with the heap) | 512 KB internal + 8 MB octal PSRAM | 180 KB internal data RAM + 4 MB PSRAM (of 8 MB) | 180 KB internal data RAM, no PSRAM |
| Screen limit | 80x25 | 256x96 (buffers in PSRAM) | 256x96 (buffers in PSRAM) | 80x25 |
| Free RAM, desktop and Terminal open | about 109 KB at boot | 199 KB internal + 7.2 MB PSRAM with Wi-Fi up (idle desktop) | about 157 KB internal with Wi-Fi up | about 108 KB before Wi-Fi, 75 KB with Wi-Fi up |
| SSH/SFTP server | yes | yes (not tested on hardware yet) | yes | no: it needs about 160 KB of internal RAM (its 64 KB heap plus headroom) |
| Modbus RTU | up to 2 RS-485 lines from the [board configuration](../guide/board-config.md) (`rs485.N.*`) | the same (any UART: the desktop is on USB); not tested on hardware yet | the same (use UART1/UART2; UART0 is the desktop) | the same |
| Ethernet | W6100 on SPI when `eth.chip = w6100` | the same; not tested on hardware yet | the same; not tested on hardware yet | the same; not tested on hardware yet |
| Console keys | none (USB Serial/JTAG) | none (USB Serial/JTAG) | `console.uart`, `console.tx`, `console.rx`, `console.baud` | the same as ESP32 |
| Flash | 8 MB: 2 x 2.5 MB OTA slots, 2.9 MB `/fs` | 16 MB: 2 x 3 MB OTA slots, 9.9 MB `/fs` (as the ESP32) | 16 MB: 2 x 3 MB OTA slots, 9.9 MB `/fs` | 4 MB: one 2.5 MB app, 1.4 MB `/fs` (the layout of TinyDesk Shell's ESP32 firmware) |
| Software Update, `ota` | yes | yes (`update-desktop-esp32s3.json`; no official S3 release yet) | yes | no (no second slot): update by flashing |
| Resets on port open | no | no | yes, with PuTTY (use the [serial bridge](../guide/terminals.md#esp32-boards-without-resets)) | the same |
| [Pixel canvas](../api/canvas.md) | internal RAM | PSRAM | PSRAM | internal RAM |

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
   Ctrl+C for repeated reads (`td_proto_set_break_check`), `ota`.
5. `td_mb_set_serial(board_rtu_lines())`.
6. The `tinydesk` task: `hal_mux_init()`, `td_init()`, apps, Terminal
   backend, `td_step()` loop (priority above the shell).
7. A 30 s timer confirms a freshly installed firmware (OTA trial), and
   `telnet_start()` opens port 23.

## Shared components

### Board interface: `link.h`

```c
bool link_init(void);
int link_read(uint8_t *buf, int cap);        /* bytes received, 0 if none; never blocks */
int link_write(const uint8_t *buf, int len);  /* bytes accepted */
const char *board_platform(void);             /* "ESP32-S3 (USB Serial/JTAG)" */
const char *board_hostname(void);             /* the shell's host name */
const td_mb_serial_t *board_rtu_lines(void);  /* Modbus RTU lines, or NULL */
```

A new ESP32-family board needs only these functions (or one of the two
existing links: `link_usj.c` for a chip's built-in USB Serial/JTAG port,
`link_uart.c` for a UART), plus its `sdkconfig.defaults`, partitions and
a project whose `main/CMakeLists.txt` includes
`ports/esp_idf/app/app.cmake`; see `ports/esp32-4mb` for the smallest
example and `ports/esp32s3` for a USB board with PSRAM. `link_usj.c`
names the chip in `board_platform()`, and `board_hostname()` is the
ESP-IDF target (`esp32c6`, `esp32s3`). A write waits up to 150 ms for
room, so a terminal that reads slowly paces the board instead of losing
frames; with no terminal reading, the USB FIFO fills and frames are
dropped. Things in the shared code that depend on the chip: the UI
task's stack (`main.c`: 8 KB on Xtensa chips, the ESP32 and ESP32-S3; 5 KB on
the C6), the chip name in About (`chip_model()` in `main.c`) and the
update information's board name (`UPDATE_BOARD` in `ota_esp.c`).

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

The ESP32-S3 port uses the same `extram.lf` and malloc settings as the
classic ESP32 with PSRAM; its 512 KB of internal RAM leave about 199 KB
free on an idle desktop with Wi-Fi up. Its PSRAM is octal
(`CONFIG_SPIRAM_MODE_OCT`; the S3's default, quad, stops the firmware at
boot on R8 chips and modules).

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
| `CONFIG_ESP_CONSOLE_NONE` | UART0 may be an RS-485 line (C6) or is the desktop (ESP32); logs go to the Log Viewer |
| `CONFIG_ESP_CONSOLE_SECONDARY_NONE` (C6, S3) | nothing else writes to the USB Serial/JTAG port the desktop uses |
| `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE` | OTA trial and rollback |
| `CONFIG_ESP_HTTPS_OTA_ALLOW_HTTP` | updates from a local HTTP server |
| `CONFIG_MBEDTLS_DYNAMIC_BUFFER` | TLS buffers only as large as needed (MQTT over TLS in about 17 KB) |
| `CONFIG_FREERTOS_USE_TRACE_FACILITY`, `CONFIG_FREERTOS_GENERATE_RUN_TIME_STATS` | the Task Manager's task list and CPU % |
| `CONFIG_FREERTOS_PLACE_FUNCTIONS_INTO_FLASH` | C6: RAM for SSH; ESP32: IRAM for the drivers |
| `CONFIG_SPIRAM*`, `CONFIG_SPIRAM_ALLOW_BSS_SEG_EXTERNAL_MEMORY` (ESP32 with PSRAM, ESP32-S3) | PSRAM for malloc and static buffers |
| `CONFIG_SPIRAM_MODE_OCT` (ESP32-S3) | octal PSRAM, as on R8 chips and modules |
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
