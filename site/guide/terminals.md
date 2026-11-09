# Terminals and connections

The desktop is a stream of VT/xterm escape sequences, so any terminal with
xterm mouse reporting and UTF-8 shows it. This page covers the ways to reach
it: the board's USB or serial port (in the browser or in PuTTY), Telnet over
the network, and the SSH/SFTP shell.

## Web terminal (in the browser)

The **<a href="console/index.html">web terminal</a>** opens a board's serial
port in Chrome or Edge, with nothing to install. It is a real terminal
([xterm.js](https://xtermjs.org/)) talking to the port through Web Serial:
it answers the desktop's size queries, sends the mouse and follows the
browser window's size.

1. Close anything that holds the port: PuTTY, `idf.py monitor`, and the
   installer's dialog.
2. Choose the board you flashed. That sets the speed (921600 for the Desktop
   edition on a classic ESP32 or an ESP32-S3, 115200 otherwise; the ESP32-C6's USB port
   ignores it) and the size (80x25 on the ESP32-C6 and the 4 MB ESP32,
   the whole window elsewhere).
3. Press **Connect** and pick the port. The desktop draws itself at once.

**Bigger text:** with the size **Fit the window**, **Text** sets the font
size (14 to 32 px, remembered by the browser). A larger font gives the
desktop fewer, larger cells, so everything on it grows and stays sharp; a
fixed size (such as **80 × 24**) uses the largest font that fits instead.
In PuTTY the same is Window → Appearance → Font.

**Pictures:** the web terminal shows Sixel graphics, so a Python app's
pictures (buttons and panels at full resolution, `canvas.picture()`) look
as on an LCD panel. So does Windows Terminal (1.22 or later); PuTTY shows
them in block pixels.

DTR and RTS are released when the port opens, so an ESP32 dev board is not
reset. The buttons send keys the browser keeps for itself (F10, F6, F11,
Ctrl+Q); **Redraw** repaints the whole screen. Add `?renderer=webgl` to the
address for the faster WebGL renderer on big screens.

The installer's own **Logs & Console** (from ESP Web Tools) is a plain log
view: it prints the escape sequences as symbols and never answers the size
query, so the desktop cannot appear there. Use the web terminal instead.

## PuTTY settings

| Setting | Value |
| --- | --- |
| Connection type | *Serial*, `COMx`; speed: any value on the ESP32-C6 (USB), **921600** on the classic ESP32 and the ESP32-S3 (UART0), **115200** for TinyDesk Shell on them |
| Window | 80x25 or larger; resize freely (the desktop follows, up to the board's limit) |
| Window → Translation | remote character set **UTF-8** |
| Terminal → Features | leave "Disable xterm-style mouse reporting" **unticked** |

Keys PuTTY handles itself: **Alt+F4** closes PuTTY (use **Ctrl+Q** to close a
TinyDesk window), **Shift+drag** is PuTTY's own text selection (copies to the
Windows clipboard), **Shift+Insert** and **Shift+right-click** paste. PuTTY
sends Ctrl+F4 as plain F4.

## Screen size

TinyDesk asks the terminal for its size every second and redraws the whole
screen when it changes. The desktop uses at most the board's limit and the
top-left corner of a bigger window:

| Board | Limit | Why |
| --- | --- | --- |
| ESP32-C6 | 80x25 | the screen buffers are internal RAM, which SSH needs |
| ESP32 with PSRAM | 256x96 | buffers in PSRAM |
| Windows and Linux programs | 400x150 | default `TD_MAX_COLS` / `TD_MAX_ROWS` |

System Monitor shows "Screen: 80x25 of 120x40 (max)" when the window is
bigger than the limit. See [Core](../api/core.md) for the size polling and
[Configuration](../api/core.md#compile-time-configuration) for the limits.

### Checking that resizing works

Open **System Monitor**: its `Terminal:` line shows the size in use, and it
changes within a second of resizing the window.

* **Windows:** start `tinydesk.exe` in **Windows Terminal**. Drag a corner of
  the window, maximise it (the desktop and its taskbar spread to the new
  edges) and press **Ctrl+-** a few times: a smaller font gives more columns
  and rows, up to 400x150. **Ctrl+0** restores the font. The old console
  window (conhost) resizes too but has no mouse.
* **Linux:** start `./tinydesk` in GNOME Terminal, Konsole or xterm and do
  the same (**Ctrl+-** / **Ctrl+Shift+-** makes the font smaller). Under WSL,
  run it in Windows Terminal.
* **ESP32 boards:** resize PuTTY or the browser window of the web terminal
  (size **Fit the window**). The ESP32 with PSRAM and the ESP32-S3 follow up to 256x96; the
  ESP32-C6 and the 4 MB ESP32 stay at 80x25 and leave the rest blank.

A maximised window beyond the limit shows the desktop in its top-left corner
only; that is the limit, not a fault.

## ESP32 boards without resets

On boards with a USB-UART chip (CP2102, CH340) the DTR and RTS lines drive
the ESP32's EN (reset) and IO0 pins so that esptool can flash it. PuTTY
raises both when it opens the port, so every connect reboots the board.
PuTTY has no option to leave the lines alone, and firmware cannot ignore a
reset. `tools/serial_bridge.py` avoids it: it opens the COM port with DTR
and RTS released, keeps it open, and lets PuTTY connect over Telnet on
`127.0.0.1`:

```bash
tools\tinydesk_putty.cmd COM10            # starts the bridge and PuTTY
python tools/serial_bridge.py COM10 --baud 921600 --listen 2310 --putty
```

or start the bridge alone and point PuTTY at *Telnet*, host `127.0.0.1`,
port `2310`. Closing and reopening PuTTY does not reset the board either:
the bridge sends the redraw request `ESC [ 5000 ~` (see
[Input](../api/input.md)) on every new connection, and the desktop comes
back as it was. Stop the bridge (Ctrl+C or close its window) before
flashing; esptool needs the port. The ESP32-C6's built-in USB port does not
have this problem.

## Remote desktop over Telnet (port 23)

After changing the root password with `passwd`, enable **Telnet desktop**
in Network. With Wi-Fi up, choose *Telnet* in PuTTY, the board's address
and port 23, and log in as **root**. Non-root accounts use SSH shell
sessions instead. Telnet is plain text: use it on a trusted network. The
Network app turns it off again (saved in NVS). Up to two clients can be
logged in at once, when both displays are Telnet; another one is told the
desktop is in use.

Which display a Telnet client gets is chosen in **Settings → Displays**
(below): by default it takes the desktop over (display 1), the board's
own screen showing a notice until the session ends, and the desktop
switches to root and back afterwards. As display 2 it is a second screen
beside the board's own instead, and the desktop stays as it is.

## Two displays

**Settings → Displays.** The desktop can be on two displays at once, like a PC with two monitors.
Each display is a terminal:

| Display | Can be |
| --- | --- |
| 1 | **This board**: its own link (USB serial, the UART, the CrowPanel's LCD); or **Telnet**: the first Telnet client to log in takes it over |
| 2 | **Off**; **Telnet**: a Telnet client (the second one when display 1 is Telnet too); or **Serial 2**: a terminal on a second serial port |

So the two can be one serial and one Telnet, two Telnet clients, or two
serial terminals. For **Serial 2**, connect a USB-serial adapter (CH340,
CP2102, FTDI; 3.3 V levels) to two spare pins and a hardware UART, set
the board keys [`display2.*`](board-config.md#second-display-serial) and
restart. (A bit-banged software UART is not offered: at the speeds a
desktop needs it would hold up the board while Wi-Fi runs.)

**Settings → Displays** (or **Displays...** in Settings and Network)
shows the two displays as boxes:

* **Duplicate these displays / Extend these displays:** both show the
  same desktop (at the smaller size of the two), or display 2 extends it.
* **Arrangement:** drag display 2 to any side of display 1 (right, left,
  above, below) and along that side, or select it and press an arrow key
  (Shift+arrows move it along the edge). The mouse and windows go across
  where the two touch.
* **Display 1 / Display 2** select the display the settings below are for:
  * **Shown on:** the terminal the display is (above); **Now:** whether
    one is connected.
  * **Resolution:** **Auto** (the terminal's window size) or a fixed size
    such as 80 x 24, 100 x 30 or 120 x 40, smaller than the terminal (the
    rest of its window stays empty).
  * **Orientation:** a terminal's window decides its own shape (make
    the PuTTY window tall for a portrait display). The CrowPanel's LCD can
    be turned: Landscape, Portrait (90°), Landscape flipped (180°) or
    Portrait flipped (270°); it applies at the next start.
  * **Make this my main display:** the taskbar, the desktop icons and new
    windows are on the main display.
* **Identify** shows each display's number on it for three seconds.

**Moving a window to the other display:** drag it across; or press
**Alt+Shift+Right / Left** (works through PuTTY), **Win+Shift+Right /
Left** (a USB keyboard on the board), or right-click its title bar →
**Other display**. Maximise, full screen and Win+Left / Right work on the
display a window is on, at that display's size.

Each display keeps its own size; both together have to fit the board's
screen memory (on an S3, 256 x 96 = 24,576 cells: for example 120 x 60
beside 200 x 60). When they do not, display 2 gets fewer columns (or rows,
above or below), and if even that is not enough, both are duplicated.
Extending needs a board with PSRAM: the C6's screen is at most 80
columns, so there the displays duplicate. Pictures (Python canvases, kiosk
artwork) go to display 1 only; display 2 sees what the windows drew under
them. A second display leaves the desktop as it is (its user, its
windows, a kiosk app): useful for watching or servicing a running
machine. The settings are kept in NVS; boards updated from 0.2.x keep
what the Network app's old Telnet setting chose.

### From another TinyDesk: Remote Desktop

**Start → Remote Desktop** shows another TinyDesk's Telnet desktop in a
window, from a board or the PC programs: its address and port (23),
**Connect**, then sign in as root in the window. Keys, mouse clicks,
the wheel and pastes go to the remote desktop; it takes the window's size
(it asks every second, so resizing or maximising the window resizes it).
**Disconnect** ends it. Its pictures are not passed on.

## SSH and SFTP

`ssh start` (or the Network app) starts TinyDesk Shell's SSH/SFTP server on port 22.
This is a shell session, not the desktop. SFTP (e.g. FileZilla) is jailed to
the user's home, root sees the whole file system. Each board has its own
host key; compare the fingerprint from `ssh hostkey` with the one the
client shows the first time. See [Commands](../shell/commands.md#ssh).

## Other terminals

| Terminal | Notes |
| --- | --- |
| Windows Terminal | desktop builds; also `telnet <board>` / `ssh`. Supports OSC 52, so the Editor's Copy reaches the Windows clipboard |
| Tera Term | Serial or Telnet; Setup → General → language UTF-8 |
| Linux/macOS terminals | `picocom -b 921600 /dev/ttyUSB0`, `screen`, `telnet`; xterm, WezTerm and kitty support OSC 52 |
| `idf.py monitor` | not for interaction: the desktop owns the port and the ESP-IDF console is off (logs are in the Log Viewer app) |
