# Using the desktop

## Keys

| Key | Effect |
| --- | --- |
| F10, or click **[Start]** | start menu |
| F6 or Alt+Tab | next window |
| Tab / Shift+Tab | next / previous widget in a window |
| F11 | full screen for the focused window (useful for `nano` in the Terminal) |
| Esc | close a dialog; with no window focused, Enter or Esc opens the start menu |
| **Ctrl+Q** (or Alt+F4 where the terminal passes it on) | close the focused window like its `[x]`; the Editor asks about unsaved changes |
| Ctrl+L | redraw the whole screen (also Start → Redraw screen) |
| arrows + Enter | move between desktop icons and open one, when no window is focused |

In the Terminal every key goes to the shell except F6, F10, F11 and Ctrl+Q.
Shift+PgUp/PgDn and the mouse wheel scroll back.

## Mouse

| Action | Effect |
| --- | --- |
| drag a title bar / bottom-right corner | move / resize |
| double-click a title bar, `[-]`, `[x]` | maximise, minimise, close |
| taskbar button | focus a window, or minimise the focused one |
| desktop icon | click selects, double-click opens |
| right-click | context menus: desktop, icons, title bars and taskbar buttons, Files entries, the Terminal, the taskbar clock |
| drag and drop | move a desktop file/folder or a Files entry onto a folder, the desktop or a Files window; onto the Terminal to type its path; onto the Editor to open it |
| hover | highlights menu items, taskbar buttons, push buttons, icons, `[-]` and `[x]` |

## Shell scripts (`.tdsh`)

Files ending in `.tdsh` are [TinyDesk Shell](../shell/commands.md) scripts.
They stand out: a green **`#!`** icon on the desktop (other files show `¶`),
and in the Files app a green name marked `#!`.

* **Right-click → Run** (on the desktop or in Files) opens the Terminal and
  runs the script there: it types `tdsh run "<path>"` and Enter, after
  clearing whatever was half-typed at the prompt. The output stays in the
  Terminal.
* Double-click (or **Open**) still opens the script in the Editor, so a
  script never runs by accident.
* From the shell: `tdsh run ~/Desktop/backup.tdsh`; a folder runs its
  `main.tdsh`.

The script language (variables, `if`/`while`/`for`, functions, pipes,
redirection, `test`) is described in [Shell scripts](../shell/scripting.md).

## Pictures (`.bmp`, `.ppm`, `.pgm`)

Pictures show the icon `▓▒` on the desktop; a double-click (or **Open**)
in Files or on the desktop opens them in the **Image Viewer**. It shrinks
or enlarges the picture to fit its window, keeping the proportions, and
draws two pixels per character cell, so its first window (60 x 20 cells)
shows at most 58 x 34 pixels: maximise the window or use full screen (F11),
or choose Fine pictures (four pixels per cell; Settings → Sizes, below) for
more detail. Larger pictures appear from top to bottom while they are read; the
desktop stays usable meanwhile.

| Key | Does |
|---|---|
| **D** | dithering on (default) or off: mixes the 256 terminal colours into smoother shades, or uses the nearest colour |
| **R** | reads the file again |
| **O** | opens another picture by its path (as the shell writes it, `~/Desktop/cat.bmp`) |

Supported: uncompressed BMP (1, 4, 8, 16, 24 and 32 bits per pixel), PPM
and PGM, binary or plain text, up to 4096 x 32768 pixels. Not supported:
PNG, JPEG, GIF and compressed (RLE) BMP: convert them on the PC first, for
example with ImageMagick (`magick photo.jpg -resize 640x480 BMP3:photo.bmp`).

To copy a picture to the board, use SFTP or FTP (Network app) or an SD
card. Without a network, a small plain-text PPM can be typed in through
the Terminal, for example with the `write` line editor or `echo`:

```text
echo 'P3 2 2 255  255 0 0  0 255 0  0 0 255  255 255 255' > ~/Desktop/four.ppm
```

```text
echo 'wificonnect; ssh start' > ~/Desktop/online.tdsh    # a script on the desktop
```

## Sizes (Settings → Sizes)

Each user picks Small / Medium / Large for three things, and the detail of
pictures:

| | Small | Medium (default) | Large |
| --- | --- | --- | --- |
| Desktop icons | one line: glyph + name | 4x3 box, two label lines | 8x5 double box, one wide label line |
| Start menu | names | glyph + name | glyph + name, two rows per item |
| Taskbar | short buttons, time only | one row | two rows: glyph + title buttons, time over date |

If the chosen icon size does not fit all icons on the screen, the next
smaller one is used; a large start menu that would not fit uses one row per
item. See [Window manager](../api/wm.md#ui-sizes).

**Pictures: Normal / Fine** is how many pixels the Image Viewer and Pixels
draw in each character cell: Normal two (one above the other, works with
every font), Fine four (two by two: twice the detail across, so thin lines
and edges get sharper). Fine needs a font with the quadrant characters
`▖ ▗ ▝ ▞ ▟`: **Cascadia Mono** (Windows Terminal's font; in PuTTY choose it
under Window → Appearance → Font), Cascadia Code or DejaVu Sans Mono.
PuTTY's default Courier New lacks them and shows small boxes along edges.
See [Pixel canvas](../api/canvas.md#picture-detail).

## Clipboard (copy and paste with the PC)

* **PC to TinyDesk:** paste in the terminal as usual (PuTTY: Shift+Insert or
  Shift+right-click). TinyDesk turns on *bracketed paste*, so the text
  arrives as one paste: the Editor inserts it as one undo step, a text box
  takes the first line, the Terminal types it into the shell. Up to 8 KB
  (`TD_PASTE_MAX`).
* **TinyDesk to PC:** in PuTTY, Shift+drag selects text and copies it to the
  Windows clipboard; Shift+Alt+drag selects a rectangle (e.g. only the
  Editor's text). The Editor's Copy/Cut also sends OSC 52, which Windows
  Terminal, xterm, WezTerm and kitty put on the PC clipboard; PuTTY 0.83
  ignores it.
* Inside TinyDesk, the Editor's Ctrl+C / Ctrl+X / Ctrl+V use a clipboard
  that is emptied when the desktop user changes.

## Users

The desktop belongs to one TinyDesk Shell user at a time: their home, Desktop
folder, settings, saved Wi-Fi networks and shell. It changes on a Telnet
login, **Start → Switch user...**, or `login` / `logout` in the Terminal; a
change closes the previous user's windows. Root can use everything; other
users are limited to their home and cannot read the system log, reboot or
set the system clock. Details: [Apps API](../api/apps.md#sessions).

## Apps

| App | What it does |
| --- | --- |
| Terminal | TinyDesk Shell in a window; the session survives closing the window |
| Files | browse, create, rename, delete, drag and drop, run `.tdsh` scripts (jailed to the home for non-root users) |
| Editor | text editor with selection, undo/redo, clipboard, paste from the PC |
| Network | Wi-Fi scan/connect/forget, Ethernet status, SSH/FTP servers, Telnet switch |
| MQTT | connect (config file or URL, TLS), subscribe, publish, message log |
| Modbus | read/write coils and registers over TCP or RTU, repeat at an interval, built-in TCP server |
| System Monitor | RAM (internal and PSRAM), tasks, CPU, frame time, serial traffic, screen size |
| Task Manager | open windows (switch to / end) and system tasks with CPU %, state, stack |
| Log Viewer | the system log (root) |
| Settings | theme (Dark by default, or Classic), ASCII mode, icons, sizes, picture detail, desktop pattern (light shade by default), links to Network / Date & time / Software update |
| Software Update | check for official updates (and, with *notify me*, daily, with a notice when one is out), install firmware from a URL or file, roll back (root); offers to save pins that only the running firmware has before it installs |
| Date & time | from the taskbar clock: clock, time zone, SNTP |
| Counter, About | demo app; version, chip, heap, uptime and public source repository URLs |
| Pixels | the [pixel canvas](../api/canvas.md) demo: an animated plasma and the palette, with pictures per second; Space switches, P pauses |
| Image Viewer | BMP, PPM and PGM pictures, scaled to fit the window ([below](#pictures-bmp-ppm-pgm)) |
