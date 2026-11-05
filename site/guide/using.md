# Using the desktop

## Keys

| Key | Effect |
| --- | --- |
| F10, or click **[Start]** | start menu |
| F6 or Alt+Tab | next window (Alt+Shift+Tab: the previous one) |
| Tab / Shift+Tab | next / previous widget in a window |
| F11 | full screen for the focused window (useful for `nano` in the Terminal) |
| Esc | close a dialog; with no window focused, Enter or Esc opens the start menu |
| **Ctrl+Q** (or Alt+F4 where the terminal passes it on) | close the focused window like its `[x]`; the Editor asks about unsaved changes |
| Ctrl+L | redraw the whole screen (also Start → Redraw screen) |
| arrows + Enter | move between desktop icons and open one, when no window is focused |

In the Terminal every key goes to the shell except F6, F10, F11 and Ctrl+Q.
Shift+PgUp/PgDn and the mouse wheel scroll back.

### Windows-key shortcuts (USB keyboard)

A keyboard plugged into the board itself ([USB keyboard and
mouse](#usb-keyboard-and-mouse-esp32-s3)) adds the shortcuts a PC desktop
has. They only come from that keyboard: the PC never sees its keys, and a
terminal such as PuTTY cannot send the Windows key, so nothing is taken
from the PC's own Win+... and Alt+Tab. F6 and the other keys above work
as before.

| Key | Effect |
| --- | --- |
| Windows key (pressed and let go alone), Ctrl+Esc | start menu |
| Alt+Tab / Alt+Shift+Tab | next / previous window |
| Alt+F4 | close the focused window, like Ctrl+Q |
| Win+Up | maximise |
| Win+Down | restore a maximised window, otherwise minimise |
| Win+Left / Win+Right | fill the left / right half of the screen |
| Win+D | minimise every window; again: bring them back |
| Win+E | Files |
| Win+R | Terminal |
| Win+I | Settings |

The window shortcuts do nothing while a dialog is open.

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
They stand out: a green **`#!`** icon on the desktop (other files show `¶`,
and [Python scripts](../shell/python.md) `py`), and in the Files app a green
name marked `#!`.

* **Right-click → Run** (on the desktop or in Files) opens the Terminal and
  runs the script there: it types `tdsh run "<path>"` and Enter, after
  clearing whatever was half-typed at the prompt. The output stays in the
  Terminal. A `.py`, `.mpy` or `.tdapp` file has **Run** too, with `py "<path>"`.
* In the Editor, **F5** saves the script and runs it with the output below
  the text (see [The Editor](#the-editor)).
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

Each user picks Small / Medium / Large for four things, and the detail of
pictures:

| | Small | Medium | Large |
| --- | --- | --- | --- |
| Desktop icons | one line: glyph + name | 4x3 box, two label lines (default) | 8x5 double box, one wide label line |
| Start menu | names | glyph + name (default) | glyph + name, two rows per item |
| Taskbar | short buttons, time only | one row (default) | two rows: glyph + title buttons, time over date |
| Windows | `[-][x]` buttons (default) | wider `[ - ][ x ]` buttons and resize corner | the same with a two-row title bar |

Larger window frames are easier to hit with a mouse on a big screen.
Windows grow by the title bar's extra row, so what is inside keeps its
size.

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

## USB keyboard and mouse (ESP32-S3)

An ESP32-S3 board can read a keyboard and mouse itself, on the chip's own
USB port (the board's second USB-C socket, marked "USB" rather than
"UART"). The terminal on the PC still shows the desktop; the board draws a
**▲ pointer** for the USB mouse, and its wheel works.

* Turn it on with `usb.host = 1` in the [board
  configuration](board-config.md#usb-keyboard-and-mouse-esp32-s3) and
  restart.
* **One device:** ESP-IDF 5.3 has no USB hub support, so use a keyboard
  and mouse set with one receiver (tested: a Dell wireless set). A hub or
  two separate devices do not work yet.
* **5 V for the device:** the port must power what is plugged in. On the
  ESP32-S3-DevKitC-1 and similar boards, close the solder pads marked
  **USB-OTG** (they connect the board's 5 V to the USB socket).
* The PC's keyboard and mouse keep working at the same time.

Two apps appear on the desktop while `usb.host = 1`; changes apply at once
and are saved in `/etc/board.conf`. Changing them needs the System
settings right (root has it; see [Accounts](first-login.md#accounts)):

* **Mouse:** pointer speed, acceleration (fast moves go further) and
  precision (slow moves go less far, for aiming at small things), each with
  a bar to click or drag and `-` / `+` buttons, and targets to try them on.
* **Keyboard:** the layout (US, UK, German, French, Spanish; AltGr works
  in those that use it), the repeat delay and rate, and a text box to try
  them in. Caps Lock and Num Lock work as on a PC.

## Kiosk mode

A board can run one [Python `tdui` app](../shell/python.md#windows-tdui)
on the whole screen instead of the desktop, for a machine's operator panel:
no taskbar, Start menu, desktop icons or window frames, and no keys that
switch, close or resize windows (F6, F10, F11, Alt+Tab, Ctrl+Q, Alt+F4 and
the Windows-key shortcuts do nothing). If the app ends or fails, it is
started again after 5 seconds. It works on every board, over the serial
link as well as on the CrowPanel's screen.

```text
# board set kiosk.app /root/hmi.py      # or a compiled .mpy or a .tdapp
# reboot
```

The [Sign in](first-login.md#signing-in) window comes first; the app starts
once an account with the Apps / Terminal right has signed in, or at once
with a [remembered login](first-login.md#remembered-login). An account
other than root cannot run an app under `/root`: put it in that account's
home, or in `/shared`. Signing out stops the app. While the app starts, a
status line or, with `kiosk.splash`, a picture fills the screen
([Board configuration](board-config.md#kiosk-mode)).

**The way out** asks for root's password and then leaves kiosk mode until
the next restart (the app goes on running, in a normal window):

* hold the mouse button (or a finger) on the **top-right corner** for 5
  seconds;
* or press **F10 three times** within 2 seconds;
* or **Ctrl+Alt+Delete** on a keyboard plugged into the board (a PC keeps
  that key for itself).

The password window has large keys to click or touch (the [on-screen
keyboard](#on-screen-keyboard)'s, with **Unlock** as Enter and **Clear**);
a keyboard types into it too. After three wrong passwords it waits 30
seconds.

**Maintenance start:** press RESET, let go, and within 1.5 seconds hold
the **BOOT** button for a moment. The board then starts without kiosk mode,
this time only (held through the reset, BOOT starts the chip's download
mode instead). Turn kiosk mode off for good with `board unset kiosk.app`.
See [Board configuration](board-config.md#kiosk-mode).

## Recycle Bin

Deleting a file or folder on the desktop or in Files (**Delete** in the
right-click menu, or the Del key in Files) asks *"Move ... to the Recycle
Bin?"* and moves it there, so it can be put back. **Delete for good** in
the same menus (Shift+Del in Files) deletes it at once, as before. The
shell's `rm` always deletes at once.

Open **Recycle Bin** (`\/` on the desktop) to see what is in it: name,
the folder it came from, when it was deleted and its size, newest first.

* **Restore** (or Enter, or a double-click) puts the selected item back
  where it was. Folders that are gone are made again; if the name is taken
  by now, it comes back as `name (2).txt`.
* **Delete** deletes the selected item for good, **Empty bin** everything.
* **Size** is how much the bin holds: Off, 256 KB, 1 MB (the default),
  4, 16 or 64 MB. When it is full, the oldest items are deleted to make
  room; something bigger than the bin is offered to be deleted for good
  instead. Off: deleting deletes for good.
* **Delete after** empties out items older than 1, 7, 30 or 90 days
  (never, by default), once the board's clock is set. It is checked when
  the desktop starts, every hour and after each delete.

Each user has their own bin, `~/.trash`, and their own size and time
(saved with their settings). A deleted item only moves within the file
system, so the space is freed when it leaves the bin, not before; on a
board with little flash, keep the size small or empty the bin now and
then. A user other than root only ever restores into their home. Files
are kept as `~/.trash/<id>/<name>` with a small `.tdtrash` record next to
them (see [Files and settings](../shell/config-files.md#trash)).

## Desktop icons

Icons fill the desktop's columns from the top left, and the Recycle Bin
sits in the bottom-right corner. Drag an icon (an app's too) to another
place and it stays there; dropped on another icon, the two swap. Moving
one turns **Auto arrange icons** off (right-click the empty desktop), so
every icon keeps its place, also after a restart: the places are saved
for each user in `~/.tinydesk_icons`. Tick **Auto arrange icons** again
to line them up. A file dropped on a folder icon still goes into that
folder.

## Shortcuts, the Start menu and Apps

Right-click a file or folder in Files and choose **Send to Desktop**: a
shortcut to it appears on the desktop (an `->` icon). Opening a shortcut
opens what it points to; deleting it deletes only the shortcut. A
shortcut is a small text file `NAME.tdlink` whose line `target=` holds
the shell path, such as `target=/root/Apps/hmi.tdapp`.

App packages (`.tdapp`) also have, in Files and on the desktop:

* **Add to Start menu** (or **Remove from Start menu**): the package
  appears in its own section of the Start menu, under the built-in apps.
  These entries are the shortcuts in the folder `~/Start menu`; rename or
  delete them there in Files.
* **Copy to Apps**: copies the package into `~/Apps`, the folder for each
  user's apps, and adds it to the Start menu. The original stays where it
  was (a kiosk app keeps working from its old path).

Both folders are made at sign-in, beside `~/Desktop`.

## Store

**Start → Store** lists the apps of the
[tinydesk-apps](https://github.com/tinydesk-project/tinydesk-apps)
catalog: App Builder apps with their sources. Choose one to see what it is
and whether this board can run it; **Install** downloads it into `~/Apps`,
checks it (its size and SHA-256 must match the catalog; a damaged download
is thrown away) and adds it to your Start menu. An installed app shows
*(installed)*, or *(update)* when the catalog has a newer version:
**Update** replaces it. **Remove** deletes it, the folder it unpacked
into and its Start menu entry. Apps a board cannot run (they need a newer
TinyDesk, a board with PSRAM, or native code for another CPU) are listed
in grey with the reason, and cannot be installed. Installing and removing
need the Apps permission. The board needs the network; the catalog is
read over HTTPS. To use another catalog, such as your own while testing,
set the board key [`store.url`](board-config.md#store).

## On-screen keyboard

For touch screens, **Start → On-screen keyboard** opens large keys at the
bottom of the screen (or above the text being typed, so it stays in
view). They type into the window that has the focus, also into dialogs;
the keyboard itself never takes the focus. A key types when the finger
lifts on it. **Shift** works as on a phone: one tap capitalises the next
character only, two quick taps lock it (**CAPS**) and one more tap
releases it. A finger between two keys still lands on one: the gaps
belong to the keys. **#+=** shows the symbols, **Esc** closes dialogs and
**Hide** closes the keyboard. Settings → **On-screen keyboard for text
fields** opens it whenever a text box has the focus and closes it again
afterwards. Sign in and the kiosk's way out use the same keys.

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

On the boards the desktop starts behind a full-screen **Sign in** window:
choose an account and type its password (**Keyboard** shows touch keys).
After 5 wrong passwords it waits 10 seconds before each try. Root can
turn on a remembered login so the board starts without it. See [First
login and users](first-login.md#signing-in).

The desktop belongs to one TinyDesk Shell user at a time: their home, Desktop
folder, settings, saved Wi-Fi networks and shell. It changes on a Telnet
login, **Start → Switch user...**, **Start → Sign out**, or `login` /
`logout` in the Terminal; a change closes the previous user's windows.
Root can use everything. Other accounts are an Admin or a User with
rights (Network, Clock, Apps / Terminal...) and stay in their home; they
cannot read the system log or install firmware. The **Users** app makes
and changes accounts ([Accounts](first-login.md#accounts)). Details:
[Apps API](../api/apps.md#sessions).

## The Editor

Double-click a file on the desktop or in Files, or drop it on the Editor.
It edits one file of up to 16 KB at a time (bigger files open read-only).

| Key | Effect |
| --- | --- |
| arrows, Home/End, PgUp/PgDn, Ctrl+Home/End, Ctrl+Left/Right | move; hold Shift to select (or drag, Shift+click, double-click a word) |
| Ctrl+A / Ctrl+C / Ctrl+X / Ctrl+V | select all, copy, cut, paste (also Ctrl+Insert, Shift+Delete, Shift+Insert) |
| Ctrl+Z / Ctrl+Y | undo / redo |
| Enter | new line with the same indent |
| Tab / Shift+Tab | indent / unindent the selected lines; without a selection, spaces to the next multiple of 4 |
| Ctrl+/ | comment out the lines with `#`, or uncomment them |
| Ctrl+F, F3 | find, find the next match |
| Ctrl+R | replace all (one undo step) |
| Ctrl+G | go to a line |
| Ctrl+S | save (a new file is saved on the Desktop) |
| Esc, Ctrl+W, Ctrl+Q | close; asks about unsaved changes |

The mouse wheel scrolls 3 lines; **Ctrl+wheel** scrolls half a page (in
the output pane too). The right-click menu has the same commands, and
turns the line numbers on or off.

### Python files

A file ending in `.py` gets more:

* **Colours** for keywords, built-ins, strings, comments, numbers and the
  names after `def` and `class`, line numbers, and the matching bracket at
  the cursor underlined.
* **Indents:** Enter adds 4 spaces after a line ending in `:` and takes 4
  away after `return`, `pass`, `break`, `continue` or `raise`; Backspace in
  the indent goes back 4.
* **F5 runs the file** (after saving it) with `py` in the Terminal's shell,
  and shows that session in an **output pane** under the text. **F7** moves
  the keyboard into the pane, for `input()` and Ctrl+C; F7 or Esc moves it
  back. Shift+F5 (or **Stop** in the menu) sends Ctrl+C from the text.
  Drag the pane's title row to resize it; the mouse wheel scrolls it back.
  `.tdsh` scripts run with F5 too.
* **The error line:** when the script stops with a traceback, the line is
  marked red in the line numbers and the message is shown in the status
  bar; **F8** goes there (opening the file it is in, if that is another
  one, such as a module in `/lib`).
* **Suggestions:** Ctrl+Space lists the names that complete the word at the
  cursor: keywords, built-ins, the modules `py` has, the names of a module
  after `module.` (`tinydesk.`, `os.`, `time.`, `asyncio.`...), and the
  words already in the file. Up/Down choose, Enter or Tab insert, Esc
  closes. With **Auto-complete** ticked in the right-click menu they pop
  up while typing; the choice is saved with the user's settings.

The pane is the same session as the Terminal window, so what the script
printed is also there, and a command typed in the pane runs in the shell.
Shift+F5 needs a terminal that sends Shift with function keys (PuTTY does
not); **Stop** in the menu and Ctrl+C in the pane work everywhere.

## Apps

| App | What it does |
| --- | --- |
| Terminal | TinyDesk Shell in a window; the session survives closing the window |
| Files | browse, create, rename, delete, drag and drop, run `.tdsh` scripts and Python files (`.py`, `.mpy`, `.tdapp`) (jailed to the home for non-root users) |
| Editor | text editor with selection, undo/redo, clipboard, paste from the PC, find and replace, go to a line; for Python, colours, line numbers, run with F5 and output below, the error line, suggestions (see [The Editor](#the-editor)) |
| Network | Wi-Fi scan/connect/forget, Ethernet status, SSH/FTP servers, Telnet switch |
| MQTT | connect (config file or URL, TLS), subscribe, publish, message log |
| Modbus | read/write coils and registers over TCP or RTU, repeat at an interval, built-in TCP server |
| System Monitor | RAM (internal and PSRAM), tasks, CPU and, on the ESP32-S3 and ESP32-C6, the chip's temperature, frame time, serial traffic, screen size |
| Task Manager | open windows (switch to / end) and system tasks with CPU %, state, stack |
| Log Viewer | the system log (root) |
| Settings | theme (Dark by default, or Classic), ASCII mode, icons, the on-screen keyboard for text fields, sizes, picture detail, desktop pattern (light shade by default), links to Network / Date & time / Software update |
| Users | accounts: new account, role (Admin or User) and rights, change password, password hint, remembered login (root); others see only their own account ([Accounts](first-login.md#accounts)) |
| On-screen keyboard | large keys to touch that type into the window that has the focus; the Start menu entry opens and closes it ([below](#on-screen-keyboard)) |
| Store | install, update and remove apps from the tinydesk-apps catalog ([above](#store)) |
| Recycle Bin | what was deleted on the desktop and in Files: restore, delete for good, empty, size and time limits ([above](#recycle-bin)) |
| Mouse, Keyboard | only with a USB keyboard and mouse on the board: pointer speed, keyboard layout and repeat; changes need the System settings right ([above](#usb-keyboard-and-mouse-esp32-s3)) |
| Software Update | check for official updates (and, with *notify me*, daily, with a notice when one is out), install firmware from a URL or file, roll back (root); offers to save pins that only the running firmware has before it installs |
| Date & time | from the taskbar clock: clock, time zone, SNTP; setting the clock and automatic time needs the Clock right |
| Counter, About | demo app; version, chip, heap, uptime and public source repository URLs |
| Pixels | the [pixel canvas](../api/canvas.md) demo: an animated plasma and the palette, with pictures per second; Space switches, P pauses |
| Image Viewer | BMP, PPM and PGM pictures, scaled to fit the window ([below](#pictures-bmp-ppm-pgm)) |
