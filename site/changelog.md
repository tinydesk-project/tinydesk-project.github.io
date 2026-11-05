# Changelog

Versions of the TinyDesk firmware and programs (`PROJECT_VER`, `TD_VERSION`),
newest first; TinyDesk Shell has its own `CHANGELOG.md`. Every board and PC
program of a release has the same version. Add an entry for every change
that reaches users or changes an API (see [Maintaining these docs](maintaining.md)).
Versions follow [Semantic Versioning](https://semver.org/): until 1.0.0 the
API may still change between minor versions.

## Unreleased

* **Password hints:** an account can have a hint, set with `useradd`,
  `passwd` or the Users app (when it is made or its password changes) and
  shown by the new **Hint** button of Sign in. A hint that contains the
  password is refused. TinyDesk Shell keeps it in NVS (`hints`). While
  root has the factory password, its hint is **Type: TinyDesk**.
* **On-screen keyboard:** one set of large touch keys, sized to the
  screen, for Sign in, the kiosk's way out and a new **On-screen
  keyboard** app that types into the focused window, also dialogs. Settings
  can open it whenever a text box has the focus. Windows can be tools that
  stay on top without taking the focus (`TD_WIN_NO_FOCUS`).
* **Desktop icons:** drag any icon to a new place (or onto another to swap
  them); the places are kept per user (`~/.tinydesk_icons`). **Auto arrange
  icons** (right-click the desktop) lines them up again. The Recycle Bin
  sits in the bottom-right corner.
* **Shortcuts and the Start menu:** **Send to Desktop** in Files makes a
  shortcut (`.tdlink`); app packages can be added to the Start menu, which
  lists a user's entries (`~/Start menu`) under the built-in apps, and
  copied to `~/Apps`, each user's folder for app packages.

* **CrowPanel 7.0 port:** `ports/crowpanel7` runs the desktop on the
  Elecrow CrowPanel 7.0 (ESP32-S3-WROOM-1-N4R8, 7" 800x480 LCD, GT911
  touch). The touch screen is the mouse (a long press is the right
  button), UART0 is the keyboard and also shows the picture
  (`lcd.mirror`), `lcd.font` picks 100x30, 80x24, 66x20 or 50x15 cells,
  and `console.uart = -1` leaves UART0 to a kiosk app. 4 MB flash, one
  app, no OTA; build it from source. See [ESP-IDF
  ports](ports/esp.md#crowpanel-70) and [Board
  configuration](guide/board-config.md#crowpanel-70).
* **Sign in, roles and rights:** the boards start behind a full-screen
  Sign in window (account list, password, touch keys; 10 seconds' wait
  after 5 wrong passwords) and the Start menu has **Sign out**. Accounts
  other than root are Admin or User with seven rights (Manage accounts,
  System settings, Network, Hardware / I/O, Shared files, Clock, Apps /
  Terminal); Admins grant only what they hold and never reach `/root` or
  other homes. The new **Users** app and the shell's `useradd NAME
  admin|user`, `userrole` and `userrights` manage them; `/shared` is a
  folder for accounts with Shared files. A changed password ends the
  account's open sessions. See [First login and
  users](guide/first-login.md).
* **Remembered login:** root can let the board start as one account
  without Sign in: the **Remember root on this device** box, the Users
  app or `bootuser NAME`. It starts only after that account signed in
  once with its password and ends with a password change;
  `bootuser --require-password` turns it off. New `td_user_ops_t`,
  `td_session_has_right()`, `td_session_lock()`, `td_login_open()` and
  `td_wm_set_input_gate()`.
* **Kiosk artwork:** `kiosk.splash` shows a picture while the kiosk app
  starts and `kiosk.status = 0` hides the status line
  (`TINYDESK_KIOSK_SPLASH` and `TINYDESK_KIOSK_HIDE_STATUS` in the PC
  programs; `kiosk_splash` and `kiosk_hide_status` in `td_sysinfo_t`).
  The kiosk app starts after Sign in.
* **Fonts in Python windows:** `tdui.Font.load()` loads a generated
  `.tdfont` bitmap font and `cv.font_text()` draws text in it at the
  screen's own pixels, aligned and wrapped in a box of cells; buttons and
  pictures take `font=`. New `td_font.h`. See [Python
  scripts](shell/python.md#fonts).
* **Gradient fills:** `rect()`, `circle()`, `bar()` and `cv.button()` take
  `gradient=` (two RGB colours, down or across); `tdui.SUPPORTS_GRADIENT_FILLS`.
* **RGB pictures:** on boards with PSRAM and on the PC,
  `tdui.enable_rgb_pictures()` keeps images, gradients and pictures in
  RGB565 instead of the 256 colours; the CrowPanel draws them directly.
  `tdui.SUPPORTS_RGB_PICTURES`, `MAX_PICTURES`, `MAX_VISIBLE_PICTURES`
  and `graphics_info()`; in C `td_picture_add_rgb()`, the picture sinks
  and `td_picture_stats()`.
* **App packages:** `py FILE.tdapp` and **Run** in Files and on the
  desktop start a `.tdapp` package made with TinyDesk App Builder; it
  checks `tdui.API` and `tdui.CAPABILITIES`, verifies its files and
  unpacks them into a `.tdapp-<hash>` folder beside it. See [Python
  scripts](shell/python.md#app-packages-tdapp).
* **tdui version:** `tdui.API` (`(1, 0)`) and `tdui.CAPABILITIES` tell
  scripts and app packages which tdui contract the firmware has, and
  `ports/common/python/tdui_contract.json` writes it down (request,
  widget and event numbers, limits, capabilities, the compatibility rule)
  for app makers such as TinyDesk App Builder; a test checks it against
  the code.
* **Bounded artwork reuse:** RGB PSRAM apps reuse losslessly compressed
  native artwork and warm hidden pictures while idle. App Builder 0.10
  exposes Automatic/Off/64/128 KiB export choices. Existing RGB packages
  benefit without re-exporting; allocation can be skipped when memory is low.
* **CrowPanel scanout under load:** UI rendering and LCD painting now share
  the RGB interrupt's CPU, avoiding simultaneous bulk pixel copies across
  both CPUs. The tested 24 MHz clock and safe framebuffer handoff remain.
  Optional `tdui.scanout_info()` distinguishes missed frame preparation from
  image-allocation or UART failures; native cache statistics expose reuse
  and drawing time independently.

* **Clean button skins:** the sample UI skin loads without dithering, and its
  keyed corners no longer bake in a fixed background. Sixel pictures keep
  the original canvas background instead of sampling the already drawn
  block fallback. These small background regions count against pixel
  budgets and are shared by page copies.
* **Button events in TinyDesk App Builder:** canvas buttons support press,
  release, click and cancel callbacks, including release outside the
  canvas. TinyDesk App Builder edits their Python bodies and starter
  actions directly, supports optional normal/pressed sprite regions, and
  exports the event wiring.

* **Atomic Python drawing:** `with cv.batch():` / `tdui.batch()` prevents
  intermediate frames across queued drawing calls, with nested groups and
  a 1000 ms recovery deadline. `cv.bar()` repaints track, fill and highlight
  in one request; instrument panels use it instead of erasing each bar first.
  Canvas buttons draw atomically and belong to their creation page.
* **App Builder pixels:** drawing controls and exports for shapes,
  gradients, pixel text, images, nine-slice panels, pictures, canvas buttons
  and bars; cell/pixel budget checks include pages and image masks. Images
  and buttons are created once and reused. The current API references and
  export fixtures are refreshed; 80×24 is a screen preset.

* **Pictures at the screen's own resolution:** a Python app's
  `canvas.picture()` (and buttons from a sprite sheet) are drawn by the
  terminal in real pixels, as Sixel graphics: smooth corners, gradients and
  sharp labels, like an LCD panel's UI. TinyDesk asks every terminal
  whether it has Sixel and how big its cells are; the web terminal (with
  xterm.js's image add-on), Windows Terminal and the CrowPanel's LCD show
  them, PuTTY shows the same in block pixels. A picture is sent once and
  again only when something drew over it. New `td_picture.h` (Sixel
  encoder, picture list), a DCS hook and Sixel answers in `td_vterm_t`, and
  `tools/make_ui_skin.py` for a skin of glossy buttons, tabs and panels.
* **Full-size file system:** a board whose LittleFS was made for a
  smaller partition (after the Shell firmware, say) now grows it to the
  whole partition at start, keeping its files; an ESP32-S3 used 1.9 MB of
  its 9.9 MB.
* **Bigger text in the web terminal:** a **Text** size (14 to 32 px) for
  the size **Fit the window**, kept by the browser, and an **80 × 24**
  size. The desktop gets fewer, larger cells, so all text grows and stays
  sharp. See [Terminals](guide/terminals.md#web-terminal-in-the-browser).
* **Round shapes look the same on both sides:** a rounded rectangle on a
  Python canvas (`rect(..., radius)`, `button()`) is rounded in whole
  cells, so its left and right ends match with Normal pictures (one pixel
  column per cell shown) as with Fine.
* **Recycle Bin:** deleting on the desktop or in Files moves the file or
  folder to the user's Recycle Bin (`~/.trash`), and the new **Recycle
  Bin** app restores it where it was (making missing folders, and
  `name (2)` when the name is taken), deletes it for good or empties the
  bin. The bin has a size limit (off, 256 KB to 64 MB, 1 MB by default;
  the oldest items go first) and can delete items after 1, 7, 30 or 90
  days; both are saved per user (bytes 17 and 18 of the settings file).
  **Delete for good** in the menus (Shift+Del in Files) deletes at once.
  New `td_trash_*()` in `td_apps.h` and `td_date_time()`.
* **USB keyboard and mouse on the ESP32-S3:** with `usb.host = 1` in the
  [board configuration](guide/board-config.md#usb-keyboard-and-mouse-esp32-s3),
  the board reads a keyboard and mouse set (one USB receiver; no hubs yet)
  on the chip's own USB port, next to the terminal's keyboard and mouse.
  The desktop draws a ▲ pointer for it, and the wheel works. The new
  **Mouse** app sets the speed, acceleration and precision, and the
  **Keyboard** app the layout (US, UK, German, French, Spanish) and key
  repeat; both are only there with such a keyboard and mouse. Ports
  provide them through the new optional `mouse` and `keyboard` in
  `td_sysinfo_t`.
* **Windows-key shortcuts:** from a USB keyboard on the board, the
  Windows key opens the start menu, Win+arrows maximise, minimise and snap
  windows to half the screen, Win+D shows the desktop, Win+E, Win+R and
  Win+I open Files, the Terminal and Settings, and Alt+Shift+Tab goes to
  the previous window. A PC terminal cannot send these keys, so they never
  clash with the PC's own; F6 and the other keys are unchanged. The input
  parser has a new modifier, `TD_MOD_SUPER`, and reads `ESC [ 27 ; m ; c ~`
  and `ESC [ 1 ; m Z`.
* **Window frame sizes:** Settings → Sizes → Windows makes the title bar
  buttons and the resize corner wider (Medium) and the title bar two rows
  high (Large), for the mouse on big screens; Small stays the default.
  New `td_wm_set_window_size()`, `td_wm_window_size()` and
  `td_wm_title_rows()`, plus `td_wm_screen_size()` and
  `td_wm_set_pointer()`.
* **Ctrl+wheel in the Editor** scrolls half a page instead of 3 lines.
* **Kiosk mode:** with `kiosk.app` in the [board
  configuration](guide/board-config.md#kiosk-mode), a board runs one
  Python `tdui` app on the whole screen instead of the desktop, on every
  board and over the serial link: no taskbar, Start menu, icons or window
  keys, and the app starts again if it ends. Root's password leaves it
  (asked after a 5-second hold on the top-right corner, F10 three times or
  Ctrl+Alt+Delete); holding BOOT just after a reset starts the board
  without it once. New `td_wm_set_kiosk()`, `kiosk_app` in `td_sysinfo_t`,
  `td_kiosk_start()` and `td_pyui_running()`; `TINYDESK_KIOSK` in the PC
  programs; `hold=` in tdsim.
* **Sure clicks with a USB mouse:** a press holds the pointer on its cell
  until the mouse has moved a whole cell, so a hand that nudges the mouse
  while clicking no longer moves the release off the button.
* **Pictures and pixels in Python windows (`tdui`):** a canvas has 2 x 2
  pixels to a cell under its text. One call draws a rectangle (with round
  corners), line, circle, dithered gradient, an image, part of one (a
  sprite sheet) or a nine-slice panel of any size, and text with the
  background `tdui.CLEAR` stands on the picture. `tdui.Image.load()`
  reads PPM, PGM and BMP files, scaled and with a transparent colour if
  asked; the file is decoded in C on the script's task and handed to the
  desktop whole, so no Python runs per pixel. Images and pixels share
  `tdui.MAX_PIXELS` (2 MB with PSRAM and on the PC, 16 KB otherwise).
  `draw_text()` writes text in pixels in an 8 x 16 font (ASCII and
  Latin-1) or a 3 x 5 one, 1 to 8 times as big, and `canvas.button()`
  makes buttons drawn in pixels, rounded in colours or from a sprite sheet,
  that look pressed while held. `tools/make_ui_sheet.py` makes an example
  sprite sheet of buttons, a panel and an LED, used by
  `examples/python/pixels_demo.py`. See
  [Pictures and pixels](shell/python.md#pictures-and-pixels).
* **The settings file grows to 20 bytes:** the picture detail moves to
  byte 16 (see [~/.tinydesk_settings](shell/config-files.md#tinydesk_settings)).
* **Fast Python screens:** a `tdui` canvas can keep up to 16 pages on the
  desktop (`w.canvas(..., pages=n)`): a script draws into a hidden page
  with `draw_to()` and shows it whole with `show()`, so switching screens
  is one step instead of a redraw that appears piece by piece;
  `copy_page()` copies a picture. `canvas.big()` draws big characters
  (digits, letters, `- + % / . :`) on the desktop in one call, and
  `put()`/`fill()` cost less Python time. On an ESP32-S3 a full-screen
  100x30 HMI now switches screens in 33–82 ms instead of 60–310 ms.
* **Bigger Python canvases:** a script's `tdui` canvases may now cover
  32000 cells in all, every page counted, on boards with PSRAM (the
  ESP32-S3 and the ESP32 with PSRAM) and in the PC programs, enough for
  ten full-screen 100x30 pages; still 2000 on the ESP32-C6 and the 4 MB
  ESP32. `tdui.MAX_CELLS` tells a script the limit of the board it runs
  on.
* **The chip's temperature in System Monitor:** on the ESP32-S3 and the
  ESP32-C6, the CPU line also shows the chip's own temperature from its
  built-in sensor (for example `Chip: 47.3 °C`). The classic ESP32 has no
  such sensor. Ports provide it through the new optional `chip_temp` in
  `td_sysinfo_t`.
* **Pixel canvas:** apps can draw pixels (`td_canvas.h`): each cell
  shows two, as an upper half block with two colours, so it works in
  every terminal TinyDesk runs in and only changed cells are sent. On
  boards with PSRAM the pixels are kept there
  (`td_canvas_set_allocator()`). `td_rgb256_dither()` dithers pictures
  and gradients to the 256 colours. See [Pixel canvas](api/canvas.md).
* **Fine pictures** (Settings → Sizes → Pictures): four pixels per cell
  with quadrant characters instead of two, twice the detail across, for
  the Image Viewer, Pixels and any canvas (`td_canvas_fit()`,
  `td_canvas_detail()`). Needs a font with the quadrants, such as
  Cascadia Mono; PuTTY's default Courier New lacks them. Saved per user
  (byte 16 of `~/.tinydesk_settings`). vtshot draws block characters as
  rectangles in its SVG pictures.
* **Pixels** app: an animated plasma and the palette with pictures per
  second, to see what a terminal link can show (up to 24 full-screen
  pictures a second at 80x25 on the ESP32-S3).
* **Image Viewer** app: BMP (uncompressed, 1 to 32 bits per pixel), PPM
  and PGM pictures (binary or plain text) up to 4096 x 32768 pixels,
  scaled to fit the window and dithered to the 256 colours (D, on by
  default), with reload (R) and open (O). Double-click a picture in Files
  or on the desktop (icon `▓▒`). Pictures are read in 4 KB parts through the new
  optional `td_fs_ops_t.read_at`, so they may be larger than the RAM;
  other apps can use the decoder (`td_image_load()`, `td_image_row()`).
  See [Using the desktop](guide/using.md#pictures-bmp-ppm-pgm).
* **No lag in slow terminals (frame pacing):** after each frame the
  terminal is asked `ESC[5n`, and the next frame waits for its answer, so
  a terminal that draws slowly (PuTTY) never builds up a backlog; a key
  in PuTTY during an animation showed after 0.1 s instead of 2-5 s
  (`TD_ACK_TIMEOUT_MS`, `TD_EV_ACK`).
* **No flicker after dropped frames:** frames a slow terminal could not
  take are sent again over the picture it shows, without clearing the
  screen; only a terminal that took nothing for 3 s (`TD_LINK_LOST_MS`)
  is set up again and cleared. On the ESP32-C6 a write waits up to
  150 ms (was 20 ms), so a slow terminal paces the board instead of
  losing frames.

## 0.2.0 (2026-10-31)

* **Python scripts with MicroPython:** the new shell command `py` runs a
  script (`py report.py`), one line (`py -c ...`) or the interactive
  prompt, in the Terminal, over SSH and in the PC programs. Scripts use
  TinyDesk's files and paths, and the `tinydesk` module runs shell
  commands, publishes MQTT messages and reads or writes Modbus registers.
  The heap is allocated only while a script runs: 512 KB of PSRAM on the
  ESP32 with PSRAM and the ESP32-S3, 64 KB of internal RAM on the
  ESP32-C6, and what is free above a 32 KB reserve on the 4 MB ESP32
  (about 31 KB with Wi-Fi off). Included: `hashlib` (sha256, sha1, md5),
  `os.urandom`, `asyncio` with `select`, and precompiled `.mpy` files;
  root can also use
  `@micropython.native` and `@micropython.viper` on the ESP32, ESP32-S3
  and ESP32-C6 and in the Linux program (the S3's and the C6's firmware
  turn off the memory protection that keeps RAM non-executable for it).
  The prompt's history (Up and Down) is kept in `~/.py_history` from one
  `py` to the next. See
  [Python scripts](shell/python.md).
* **Pins and buses from Python (`machine`):** on the ESP32 boards, root's
  scripts use `Pin` (with interrupts), `ADC`, `PWM`, `I2C`, `SPI`, `UART`,
  `SoftI2C`, `SoftSPI`, `time_pulse_us`, `bitstream` and the `neopixel`
  module. Pins of the board configuration and the desktop's link are
  refused, strapping pins get a warning, and everything a script opened is
  closed and reset when it ends. The PC programs import `machine` but have
  no hardware for it. See [Hardware](shell/python.md#hardware-machine).
* **Windows from Python (`tdui`):** a script started from the desktop opens
  windows with labels, buttons, text boxes, check boxes, lists, progress
  bars and character canvases, shows message boxes, input boxes and popup
  menus, and handles clicks, keys and timers in Python: a small app without
  writing C. The desktop makes the windows on its own loop, so a busy
  script never slows it; the windows close when the script ends, and
  Ctrl+C in one of them stops it. Scripts can resize, move, maximise and
  minimise their windows, hear when the user resizes one (`on_resize`),
  and lists and canvases can follow the new size.
  See [Windows](shell/python.md#windows-tdui).
* **The Editor for Python:** a `.py` file gets colours, line numbers,
  indents after `:`, and the matching bracket shown. **F5** saves and runs
  it, with the output in a pane under the text (F7 types into it); after
  an error the line is marked and **F8** goes there. Ctrl+Space suggests
  names (keywords, built-ins, modules and their functions, the file's
  words), which can also pop up while typing (right-click →
  Auto-complete, saved per user). For every file: find (Ctrl+F, F3),
  replace all (Ctrl+R), go to a line (Ctrl+G), comment lines (Ctrl+/),
  Tab / Shift+Tab on selected lines and Enter keeping the indent.
  See [The Editor](guide/using.md#the-editor).
* **The Editor's buffer follows the file:** it is sized to the file and
  grows with the text up to 16 KB, instead of taking 16 KB for any file.
  A small script then leaves enough RAM to run it with F5 on the 4 MB
  ESP32.
* **Run `.py` files** from the desktop and the Files app (right-click →
  Run), as `.tdsh` scripts; they show a `py` icon.
* **Ctrl+C stops `modbus read -i` in the PC programs** too, as on the
  boards.
* **Pastes no longer pick up the size reply:** the desktop's periodic
  terminal size query could be answered in the middle of a bracketed
  paste, and the reply (such as `[25;80R`) ended up in the pasted text, in
  up to one paste in ten of a few KB at 921600 baud. The reply is now taken
  out of the paste and handled as usual, and the query is not sent while a
  paste is arriving.
* **ESP32-C6: long pastes arrive whole.** The USB receive buffer grows from
  1 KB to 8 KB, as on the other boards: when it filled while the desktop
  was drawing, the USB driver dropped whole 64-byte packets, and a paste
  of a few KB could lose some of its text. It costs 7 KB of RAM.
* **Ports can put their own files on new Desktops:** `td_sysinfo_t` has a
  new optional member `desktop_files`, a list of file names and contents
  that TinyDesk writes next to `Welcome.txt` when a user's Desktop folder
  is created, for example a port's `Credits.txt`. Written once per user,
  like `Welcome.txt`; builds that do not set it are unchanged. See
  [Files for a new Desktop](api/sysinfo.md#files-for-a-new-desktop)
  (tinydesk #5).
* **`CONTRIBUTORS.md`** in the `tinydesk` repository lists the people who
  made TinyDesk and TinyDesk Shell.
* **ESP32-S3 port** (tinydesk #9, by SanjayG-xls): TinyDesk Desktop on an
  ESP32-S3 with 16 MB flash and 8 MB Octal PSRAM (e.g.
  ESP32-S3-DevKitC-1 N16R8), on UART0, the board's USB-UART port, at
  921600 baud. Screens up to 256x96, two OTA slots with Software Update,
  the SSH server, and the same `console.*` board keys as the classic ESP32
  (default pins TX 43, RX 44). Project `ports/esp32s3`; CI builds it.
* **TinyDesk Shell on the ESP32-S3:** `projects/esp32s3` in TinyDesk Shell
  runs on any ESP32-S3 with 4 MB flash or more, at 115200 baud, and uses
  Octal PSRAM when the module has it. `peek` and `poke` know the S3's
  memory map ([peek, poke](shell/commands.md#peek-poke)).
* **Install:** the installer and the web terminal offer the ESP32-S3 for
  both editions; releases carry `tinydesk-desktop-<version>-esp32s3-factory.bin`
  (with its app image and update feed) and
  `tinydesk-shell-<version>-esp32s3-factory.bin`.

Known issue: `ssh start` refuses on the ESP32-C6 with "insufficient
internal RAM", as in earlier releases. A fix is tested and comes in a
later release.

## 0.1.6 (2026-10-18)

* **`peek` and `poke` on the ESP32 boards** (TinyDesk Shell): root can
  read RAM, ROM, PSRAM and the GPIO, IO_MUX and eFuse registers, and
  write the GPIO output registers; `peek -l` lists the ranges. See
  [Commands](shell/commands.md#peek-poke) (tinydesk-shell #1).
* **Scripts that run out of memory while loading say so**
  (TinyDesk Shell): `uscript: <file>:<line>: not enough memory to load
  the script` instead of returning with no output (tinydesk-shell #2).
* **TinyDesk Shell builds on macOS** (tinydesk-shell #5).
* **About** shows the TinyDesk Shell repository's address in full; it was
  cut off at the window's edge.
* **nano with UTF-8 text** (TinyDesk Shell): the cursor, Backspace and
  Delete work by character instead of by byte, so text such as `Привет`
  or `日本語` is no longer cut into broken bytes; Up and Down keep the
  column, and the column shown counts characters (tinydesk #4).
* **Editor:** a build can set the number of undo records,
  `TD_EDITOR_UNDO_OPS` (128 by default, 20 bytes each), like the text
  buffer (`TD_EDITOR_MAX`) and the undo text (`TD_EDITOR_UNDO`), to fit
  the Editor into boards with less RAM (tinydesk #6).
* **TinyDesk Shell:** ports can set the script memory limits
  (`TDSH_MAX_VARS`, `TDSH_VAR_NAME_MAX`, `TDSH_VAR_VALUE_MAX`,
  `TDSH_SCRIPT_TASK_STACK`) from the build; TinyDesk keeps the defaults.
  Every component that includes `tdsh.h` must use the same values
  (tinydesk-shell #4).

## 0.1.5 (2026-10-06)

TinyDesk 0.1.5, with TinyDesk Shell 0.1.5.

* **Update information can move:** an update feed may name its new
  address (`moved`); the board keeps it and reads the update information
  there from then on (HTTPS only; `update.url` still comes first).
  `ota feed` shows the address in use, `ota feed reset` forgets a moved
  one. Should the site move again, boards from this release on follow.
* **Board configuration numbers** are decimal or `0x` hexadecimal; a
  leading zero no longer means octal (TinyDesk Shell).
* **Only root changes the network policy:** `network mode` and
  `network autowifi` with a value now need root (TinyDesk Shell).
* **New addresses:** the code is at github.com/tinydesk-project and the
  documentation at https://tinydesk-project.github.io/. Boards from this
  release on check that site for updates.

## 0.1.4 (2026-10-05)

TinyDesk 0.1.4, with TinyDesk Shell 0.1.4.

* **Non-ASCII names line up:** Wi-Fi network names in the Network window,
  file names in Files and window titles in the Task Manager with
  characters such as é, ☕ or 中 no longer push the columns after them out
  of line (the signal bars and *secured* overlapped the name). The columns
  were padded by bytes; they are now padded by characters, as the screen
  draws them, and a long name is cut at a character boundary. New
  `td_utf8_pad()`. Double-width characters still take one cell.
* **Board settings and official updates:** firmware built with your own
  `board.conf` has its pins built in; an official update, built without
  them, would start without Ethernet, SD card or RS-485. `board save`
  copies them to `/etc/board.conf`; Software Update offers *Save and
  install* before it installs, and `ota install` refuses until they are
  saved (`-f` installs anyway). `board show` says when settings come only
  from the firmware.
* **No character cut in half:** desktop icon labels (and the glyph of an
  item without its own), window titles, text set on widgets and the
  Delete and Save questions shorten names by whole characters; a
  non-ASCII name showed a broken character or was cut too short. New
  `td_utf8_copy()` and `td_utf8_skip()`.
* **Shell: long command lines** that wrap over several rows are redrawn
  correctly while you type, delete or move the cursor (TinyDesk Shell);
  the Terminal window tells the shell its width.
* **Contributing:** a pre-commit hook formats the C files of every commit
  with clang-format 16.0.6 (`pip install pre-commit`, then
  `pre-commit install`), and tinydesk-shell has a `CONTRIBUTING.md`.
* **Editor:** a file keeps its name when it is saved. On Linux the
  name was lost after the first Ctrl+S, so the next one asked for a
  name again ([#2](https://github.com/tinydesk-project/tinydesk/pull/2)).
* **`mqtt status`** says `TLS handshake` while a TLS connection is being
  set up; it said `not connected`
  ([#1](https://github.com/tinydesk-project/tinydesk/pull/1)).
* **Tests:** `test_widgets` (text boxes, lists, scrollbars, checkboxes,
  buttons, focus, message and input boxes) and `test_editor` (undo and
  redo, cut, copy and paste, closing with unsaved changes)
  ([#2](https://github.com/tinydesk-project/tinydesk/pull/2)).
* **Source layout:** the ESP code shared by the three ESP-IDF projects
  moved out of `ports/esp32c6` into `ports/esp_idf` (the application in
  `app/`, the TinyDesk component in `components/tinydesk/`); the board
  projects keep only their link, partitions and settings. The ESP32 and
  4 MB ESP32 projects download their own third-party components instead
  of using the C6 project's (no more `IDF_COMPONENT_MANAGER=0`).
  `serve_bin.py` is now in `tools/`. The firmware is unchanged.
* **Code style:** the C sources of TinyDesk and TinyDesk Shell are
  formatted with `clang-format` (`.clang-format` in each repository):
  braces on their own lines, one statement per line. Layout only; the
  code is unchanged.

## 0.1.3 (2026-10-01)

TinyDesk 0.1.3, with TinyDesk Shell 0.1.3.

* **Official updates in Software Update** (ESP32-C6, ESP32 with PSRAM):
  *Check for official updates* reads the update feed of the newest release
  next to the web installer, puts its image into the URL field and shows
  its version, date, size and release page; *Install* installs it. With
  *Check for official updates daily and notify me* (on by default) the board
  checks 2 minutes after start-up and then daily, and says once per version
  that a newer one is out. In the shell: `ota official`, `ota notify`. The
  board key `update.url` reads another feed. Releases carry the app images
  (`tinydesk-desktop-<version>-<board>-app.bin`) and the feeds
  (`update-desktop-<board>.json`). Text boxes can hold text longer than a
  widget's own (`td_textbox_set_buffer()`, `td_widget_text()`).

Reported on Reddit with an ESP32-2432S028R (4 MB, no PSRAM):

* **Crash when opening the Log Viewer with several windows open.** The
  widget pool (shared by all windows) ran out, and the Log Viewer used the
  list it had not got. A window that does not get all its widgets, its tick
  timer or a window slot is now closed again with *Too many windows are
  open. Close one, then try again.*; the last window slot and 4 widgets are
  kept so that message can always be shown. The pools are larger: 16
  windows, 96 widgets on boards without PSRAM (was 40) and 128 with it,
  24 timers. Tested by opening all 13 apps one after another on all three
  boards.
* The cover's TinyDesk Shell picture shows a session without a version
  number (it went out of date with every release).
* **Software Update on the 4 MB ESP32** said "not available on this
  platform" and that "the ESP32 build updates itself". It now says why (one
  app slot) and how to update instead: the web installer without *Erase
  device*. A port can give that text in `td_sysinfo_t.no_ota_text`.
* **Empty Task Manager with many windows open:** the task list needed new
  memory on every refresh. It now keeps its buffers, and says *Not enough
  memory to list the tasks* when it cannot get them; the Task Manager, MQTT
  and Modbus say *Not enough memory* instead of not opening at all.
* **nano in the Terminal window:** it assumed an 80x24 screen, so in a
  smaller window its status line (where Ctrl+C shows the cursor position)
  and help lines were cut off. nano now asks the terminal for its size
  (on the boards and in the Linux program).
* **The factory password is named where it is needed:** `passwd` says that
  the old password is `TinyDesk` while root still has it, and the
  messages that keep Telnet, SSH and FTP off until it is changed name it.

## 0.1.2 (2026-09-28)

TinyDesk 0.1.2, with TinyDesk Shell 0.1.2.

* **SD card at `/sd`** (TinyDesk Shell's new `sd` command, root only):
  `sd mount`, `sd umount`, `sd format --yes`, `sd status`, and
  `sd.automount = 1` in the board configuration for mounting at boot. The
  card is `/sd` in the shell, FTP and SFTP, and the folder `sd` in Files
  (`td_fs_stdio_redirect()`). Tested on an ESP32-C6 with its SD card on the
  W6100's SPI bus; long file names work.
* **`ping` in the Terminal window** (TinyDesk Shell): the replies and the
  statistics now appear (only the `PING` line did), and `ping -c <count>`
  works on the boards as on the PC.
* **Documentation and web installer:**
  * The cover shows the current version and puts *Install / Download*
    (straight to the installer) right under the headline; `tools/fetch_release.py` writes the installed
    release's version into the cover and the installer.
  * Install: the Desktop edition's needs are given per board (the 4 MB
    ESP32 runs without PSRAM, with an 80x25 screen and no SSH server or
    OTA); updating is described per board, and the installer's advice
    follows the chosen board (no Software Update on the 4 MB ESP32 or in
    TinyDesk Shell).
  * Brighter small text on the cover and in the web terminal; the
    sidebar's section headings read like its links; "Installation" in the
    sidebar.
  * The installer shows the esptool command for the chosen board only (with
    the release's file name), and only for ESP boards.
  * First start and esptool: the SSH note and the erase advice follow the
    chosen board (no SSH server in the 4 MB Desktop build; no erase needed
    to switch editions on 4 MB boards). The cover's Shell picture is a new
    capture of TinyDesk Shell 0.1.2 on an ESP32.
  * Phones: wide tables scroll in their own box instead of being cut off,
    the text uses the full width, and the web terminal fits an 80x25 board
    screen without scrolling the page sideways.
  * Getting started and Troubleshooting: keep the checkout in a short
    folder on Windows (`ninja: error: mkdir(...)`).

## 0.1.1 (2026-09-28)

TinyDesk 0.1.1, with TinyDesk Shell 0.1.1.

* **PC programs: MQTT over TLS in the downloads.** The Windows and Linux
  programs of the 0.1.0 release were built without mbedTLS and refuse
  `mqtts` ("not available in this build"); programs built from source with
  ESP-IDF installed are not affected. CI and the release workflow now build
  against mbedTLS v3.6.7, stop without it (`TD_REQUIRE_TLS`) and test that
  TLS is built in.
* **TinyDesk Shell releases** in its own repository, from its own release
  workflow, with the same file names as the Shell edition here.
* **TinyDesk Shell for Windows: `tdsh.exe`.** A native Windows program of
  the Shell edition (`tinydesk-shell-windows-x64.zip`), for Windows
  Terminal: line editing, history and Tab completion, `ifconfig` and
  `ping`. Its files are in `%LOCALAPPDATA%\tdsh\rootfs`; the user is your
  Windows user name. The Desktop edition's Windows program now uses the same
  Windows port from TinyDesk Shell.
* **Documentation:** version numbers from before the first release removed
  from the troubleshooting page and the `ota` examples; "TinyDesk" written
  the same way everywhere.

## 0.1.0 (2026-09-28): first public release

TinyDesk 0.1.0, with TinyDesk Shell 0.1.0.

* **Desktop:** overlapping windows (move, resize, minimise, maximise, full
  screen), taskbar, start menu, desktop icons from each user's Desktop
  folder, mouse and keyboard, drag and drop, copy and paste with the PC,
  themes and UI sizes; the screen follows the terminal's size.
* **Apps:** Terminal (TinyDesk Shell), Files, Editor, Network, MQTT
  (TLS), Modbus TCP/RTU with a TCP server, System Monitor, Task Manager,
  Log Viewer, Settings, Software Update (OTA with rollback), Date & time.
* **Shell scripts:** `.tdsh` files have their own icon and a **Run** entry
  in their right-click menu.
* **Users:** per-user desktops, files, Wi-Fi networks and settings; Telnet
  remote desktop; factory root password `TinyDesk`.
* **Board configuration:** pins come from `board.conf` / `/etc/board.conf`,
  not from the code (`board` command).
* **Ports:** ESP32-C6, ESP32 with PSRAM, ESP32 with 4 MB flash and no
  PSRAM, Linux, Windows; macOS and other POSIX systems from source.
* **Editions:** TinyDesk Desktop and TinyDesk Shell, both with a web
  installer and PC downloads.
* **About** shows both source repositories (`tinydesk` and
  `tinydesk-shell`) and fits its content on every board; the name is
  spelled TinyDesk in About, Software Update, `ota status` and the welcome
  text, and free RAM is given in KB everywhere.
* **Windows:** the Terminal's shell now has `ifconfig`, `ping`, `date`,
  `cal`, `tz`, `write`, `hostpath` and `capabilities`, like the Linux
  program (`ifconfig` was "command not found").
* **Fix (Windows):** the desktop now follows the Windows Terminal window
  when it is maximised or resized. A resize now only repaints; the terminal
  setup it resent ("alternate screen on") made the Windows console fall
  back to its old size.
* **Docs:** [Shell scripts](shell/scripting.md), the `.tdsh` language
  reference (it was only shown by the shell's test scripts).
* **Web terminal** (`console/` on this site): opens a board's serial port in
  Chrome or Edge with a real terminal (xterm.js over Web Serial), presets for
  every board, no reset of ESP32 dev boards. The installer links to it; its
  own Logs & Console cannot show the desktop.
* **Dark theme by default** on every port, with a clearly visible teal
  desktop pattern (light shade). Classic stays available in Settings;
  saved settings keep their theme.
* **Windows and Linux programs:** the desktop follows the terminal up to
  400x150 (was 132x50), like the ESP32 with PSRAM does up to 256x96.
* **Build date:** the date and time shown by About, Software Update and
  `ota status` now come from the build that made the image. ESP-IDF compiled
  them only in a fresh build directory before, so incremental builds kept an
  old date (`ports/common/app_desc_stamp.cmake`).

## Development builds (before the first release)

These builds were never published; their notes are kept for reference.

### dev 0.2.17 (2026-09-26)

* **Board configuration**: pins and other hardware settings are no longer
  compiled in. RS-485 lines (`rs485.N.*`), W6100 Ethernet (`eth.*`), the
  SD card (`sd.*`) and the classic ESP32's console UART (`console.*`) come
  from `key = value` settings: the port's `board.conf` (not in git) or
  `board.example.conf` built into the firmware, overridden per device by
  `/etc/board.conf`. New shell command `board` (show, get, set, unset,
  init). API `tdsh_board.h`. See
  [Board configuration](guide/board-config.md).
* Ethernet stays off unless `eth.chip = w6100`; `lan hw set` and
  `lan poll` write the board configuration. The W6100 Kconfig pin
  defaults are now -1 (fallbacks only).
* `hwtest` runs on both ESP ports with the configured pins; unconfigured
  tests are skipped (`[SKIP]`).
* Modbus RTU on the classic ESP32 too, when RS-485 lines are configured.
* Factory root password **`TinyDesk`**, accepted everywhere (Telnet, SSH,
  FTP, Switch user). No forced change at the first start any more (the
  0.2.16 dialog and `td_sysinfo_t.root_password_is_default` /
  `set_root_password` are gone). `MicroPython` or `tinydesk` as root's
  password is switched to `TinyDesk` once.
* Shell scripts use the extension `.tdsh`; the boot script is
  `~/.tdshrc.tdsh`, history `~/.tdsh_history`, time zone `~/.tdsh_tz`, shebang
  `#!/bin/tdsh`, a directory runs `main.tdsh`.
* Fix: a board without Ethernet crashed at boot when the Telnet listener
  opened its socket before the TCP/IP stack existed (seen on the classic
  ESP32 as a reset loop). `app_main` now starts `esp_netif` and the
  default event loop first.
* ESP32: FreeRTOS functions in flash (`CONFIG_FREERTOS_PLACE_FUNCTIONS_INTO_FLASH`)
  to fit IRAM.
* The startup-file template no longer says "USB Serial/JTAG" on every board.
* The shell is now **TinyDesk Shell 0.7.0** (`tdsh`), MIT, in
  its own repository and included as the submodule `third_party/tdsh`.
  The Terminal window is titled `Terminal - tdsh`; About shows
  `Shell: TinyDesk Shell 0.7.0`.
* Docs: Board configuration, Contributing and repositories; README rewritten
  for GitHub; MIT licence for TinyDesk Shell.
* Everything that was named after the shell's origin is now `tdsh` /
  TinyDesk Shell: the submodule `third_party/tdsh`, the ESP-IDF component
  `tdsh`, `tdsh_*` functions and files, `CONFIG_TDSH_*`, `TD_WITH_TDSH`,
  the command `tdsh run`, the files `~/.tdsh_history` and `~/.tdsh_tz`, the
  SSH host key's NVS namespace `tdsh_ssh` (a board keeps a new host key).
* `.tdsh` scripts: **Right-click → Run** on the desktop and in Files runs
  the script in the Terminal (`td_terminal_run()`, `td_script_run()`); a
  green `#!` icon on the desktop (new optional `icon_fg` in
  `td_desktop_provider_t`), a green name with `#!` in Files.
* Two editions: **TinyDesk Desktop** and **TinyDesk Shell**. The installer
  page offers both for ESP32-C6, ESP32, Linux, Windows and macOS/other;
  `make_release.py` packages both editions and the PC programs
  (`--host-*`); the release workflow builds Linux and Windows programs.
* Docs search: results and the opened page highlight the searched words
  (`?highlight=`), with "Hide search matches".
* The tagline no longer says ESP32: "A windowed desktop for
  microcontrollers, drawn in any terminal" (About, docs, READMEs).
* Release builds (`-O3`) compile without warnings: a path that does not fit
  is refused instead of being cut (Files, `remove_tree`), and display texts
  are cut on purpose.
* The ESP32 port's libsmb2 MD5 shim moved into TinyDesk Shell
  (`tdsh_md5_rom.c`).

### dev 0.2.16 (2026-09-25)

* Factory root password `tinydesk` (was TinyDesk Shell's `MicroPython`; a board
  that still has that is switched to `tinydesk` once). It is accepted only
  on the board's own console; Telnet, SSH, FTP and Switch user refuse it.
* First start: the desktop asks for a new root password and cannot be used
  until one is set (`td_sysinfo_t.root_password_is_default`,
  `set_root_password`; TinyDesk Shell `tdsh_user_has_default_password()`,
  `tdsh_user_set_password()`).
* Prebuilt firmware: `tools/make_release.py` (parts, merged factory images,
  ESP Web Tools manifest, checksums), a browser installer page
  (`docs/install/`) and a GitHub Actions release workflow.
* Docs: Install, First login and users, Security, Troubleshooting; cover
  page, TinyDesk colours, light/dark switch, previous/next links, OS tabs;
  pinned CDN versions.
* `tdsim factory-root` simulates a fresh board.

### dev 0.2.15 (2026-09-25)

* `tools/serial_bridge.py` and `tools/tinydesk_putty.cmd`: PuTTY on an
  ESP32 dev board without resetting it.
* Input: `ESC [ 5000 ~` (`TD_SEQ_REDRAW`) asks for the terminal setup and a
  full redraw; the bridge sends it on every new connection.
* Documentation site (`docs/`, Docsify).
* `modbus` numbers and the Modbus app's fields are decimal unless they
  start with `0x` (a leading 0 was octal: `010` was 8).
* `modbus read -i`: the header says where Ctrl+C works; `-n 0` is refused
  where Ctrl+C cannot stop it (host builds).
* TinyDesk Shell patch: SSH messages say "PSRAM heap" on the classic ESP32.
* Header comments corrected (`td_wm_windows()` order, `td_update_register()`,
  `td_key_name()`, `td_config.h`, `td_mqtt.h`).

### dev 0.2.14 (2026-09-25)

* Task Manager app: open windows (switch to, end task) and FreeRTOS tasks
  with state, priority, core, CPU % and unused stack
  (`td_sysinfo_t.tasks`, `td_task_info_t`, `td_wm_windows()`).
* Bracketed paste: `TD_EV_PASTE`, `td_paste_text()`, `td_widgets_paste()`;
  the Editor, text boxes and the Terminal take pastes from the PC.
* OSC 52 on the Editor's Copy/Cut (`td_host_clipboard_set()`).
* Ctrl+Q (and Alt+F4) close the focused window.
* ESP32: UART RX buffer 8 KB. ESP32-C6: FreeRTOS kernel code in flash
  (`CONFIG_FREERTOS_PLACE_FUNCTIONS_INTO_FLASH`), about 14 KB more free RAM.
* TinyDesk Shell patch: a W6100 setup that fails half-way is undone (it leaked a
  task and an SPI device on every retry on boards without the chip).

### dev 0.2.13 (2026-09-24)

* The whole screen is redrawn whenever the terminal reports a new size,
  even if the size used stays the same (fixes a blank screen after resizing
  PuTTY past the C6's 80x25). Size poll every second.
* ESP32 screen limit 256x96; System Monitor shows the terminal's size when
  it is bigger than the limit (`td_stats_t.term_cols`, `term_rows`).

### dev 0.2.12 (2026-09-24)

* `modbus read ... -i ms -n times`: repeated reads with Ctrl+C
  (`td_proto_set_break_check()`, `tdsh_bridge_break_requested()`).
* Settings → Sizes: desktop icons, start menu and taskbar in Small /
  Medium / Large (`td_ui_size_t`, `td_wm_set_*_size()`), clock parts for the
  two-row taskbar (`td_clock_provider_t.parts`), window icons.
* Screen limits from `CONFIG_SPIRAM` (ESP32 160x60 at the time, C6 80x25).

### dev 0.2.11 (2026-09-24)

* Modbus app: repeat interval in ms (10 ms to 1 hour), faster polling and
  redraws only on change.

### dev 0.2.10 (2026-09-24)

* Per-device SSH host key (TinyDesk Shell patch; `ssh hostkey [new]`).
* The Modbus app's device hint matches the board's RTU lines.

### dev 0.2.9 (2026-09-24)

* TinyDesk Shell patch: `ssh stop` really stops the server (lwIP `accept()` needed
  a receive timeout).

### dev 0.2.8 (2026-09-24)

* Classic ESP32 port (`ports/esp32`, ESP32-WROVER-IE): UART0 link at
  921600, PSRAM for the static buffers, per-board `link.h`.
* Board-neutral shared ESP code (`hal_mux.c`), System Monitor shows
  internal RAM and PSRAM separately (`td_sysinfo_t.psram_free/total`).

### dev 0.2.0 - 0.2.7 (2026-09-24)

* OTA updates with rollback, Software Update app, `ota` command, new
  partition table with two app slots.
* MQTT client and app (config file, TLS 1.2 with CA/client certificates),
  Modbus TCP/RTU client, TCP server and app, `mqtt` and `modbus` commands.
* Two RS-485 channels on UART1/UART0 in RS-485 half-duplex mode; the
  ESP-IDF console is off, logs go to the Log Viewer.
* TLS memory: certificate files read at their size, dynamic mbedTLS buffers.

### dev 0.1.x (2026-09-23 - 2026-09-24)

* The core: screen buffers with diff rendering, input parser with SGR
  mouse, window manager, widgets, themes, terminal emulator, timers.
* Apps: Terminal (TinyDesk Shell), Files, Editor (undo/redo, selection,
  clipboard), Network, System Monitor, Log Viewer, Settings, Date & time,
  Counter, About; desktop icons and the Desktop folder, context menus,
  drag and drop, hover highlights, taskbar tray.
* Users: per-user desktops, settings, Wi-Fi networks, SSH/SFTP jail;
  Telnet remote desktop on port 23.
