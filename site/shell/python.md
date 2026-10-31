# Python scripts (MicroPython)

TinyDesk runs Python scripts with [MicroPython](https://micropython.org)
1.29 through the shell command `py`. It works wherever the shell does: the
desktop's Terminal window, Telnet and SSH sessions, and the Windows and
Linux programs.

```
# py hello.py                 run a script
# py hello.mpy                run a precompiled script
# py                          the interactive prompt (>>>)
# py -c "print(2 ** 100)"     run one line
```

Scripts use TinyDesk's own files and paths, and reach the rest of TinyDesk
(shell commands, MQTT, Modbus, the Log Viewer) through the `tinydesk`
module. Root's scripts drive the free pins and buses with `machine` (see
[Hardware](#hardware-machine)), and a script started from the desktop can
open windows of its own with `tdui` (see [Windows](#windows-tdui)).

## Running scripts

Usage: `py [-m KB] [FILE [ARG...] | -c CODE [ARG...]]`

| Form | What it does |
|---|---|
| `py FILE [ARG...]` | Runs the script: Python source, or a `.mpy` file (see [Precompiled .mpy files](#precompiled-mpy-files)). `sys.argv` is `[FILE, ARG...]`; the script's directory comes first in `sys.path`, so it can import modules next to it. |
| `py -c CODE [ARG...]` | Runs `CODE`; `sys.argv` is `['-c', ARG...]`. |
| `py` | The interactive prompt, with history, auto-indent and paste mode (Ctrl+E, then Ctrl+D to run). Ctrl+D or `exit()` leaves it. `help()` lists what is available. Up and Down (or Ctrl+P and Ctrl+N) recall the last 8 lines, also from earlier prompts: they are kept in `~/.py_history`, as the shell keeps its own in `~/.tdsh_history`. |
| `-m KB` | Heap size in KB (16 to 65536) instead of the default (see [Memory](#memory)). |

Exit status: 0 when the script ends normally, 1 after an uncaught
exception (the traceback is printed), 130 after Ctrl+C, and *n* after
`sys.exit(n)`. A missing script or a bad option gives 2. Scripts can be
called from `.tdsh` scripts and their status tested with `$?`.

**Ctrl+C** in the desktop's Terminal window stops a running script with
`KeyboardInterrupt`, also inside `time.sleep()` and `input()`, on every
board and in the PC programs (Telnet shows the same desktop). In an SSH
session it only interrupts `input()`; the same holds for
`modbus read -i`.

Who: any user. One script runs at a time on a device: while one runs, `py`
in another session (or from `tinydesk.shell()`) says
`py: Python is already running (one script at a time)`.

## Writing scripts on the desktop

The desktop's [Editor](../guide/using.md#the-editor) knows Python: it
colours a `.py` file, numbers its lines and indents after `:`. **F5** saves
and runs the file with `py`, with the output in a pane under the text (F7
types into it, for `input()` and Ctrl+C). When the script stops with an
error, the line is marked and **F8** goes to it. **Ctrl+Space** suggests
names: keywords, built-ins, modules, a module's functions after the dot
(`tinydesk.`, `time.`, `asyncio.`...) and the words of the file; they can
also pop up while typing (right-click → Auto-complete). On the desktop and
in Files, right-click → **Run** runs a `.py` file in the Terminal.

## Files

Paths are TinyDesk paths, the same ones the shell uses: `/home/root`,
`/etc`, `/tmp`, relative to the shell's current directory when the script
started. `open()`, `os.listdir()`, `os.stat()`, `os.mkdir()`,
`os.remove()`, `os.rename()`, `os.rmdir()`, `os.chdir()` and `os.getcwd()`
work on them. Every path goes through the shell's own path rules, so a
script reaches exactly the files the shell's commands reach, and `..`
cannot leave the TinyDesk root.

* `os.chdir()` in a script does not change the shell's directory.
* Files are always written in binary form: a `\n` stays `\n`, also on
  Windows.
* MicroPython closes a file only when the garbage collector frees it, not
  when the last reference goes away. Use `with open(...) as f:` or call
  `f.close()`, so the data is on the flash before the script goes on
  (Windows also refuses to rename or delete a file that is still open).
* `import mod` looks in the script's directory (or the current directory
  at the prompt), then among the modules built into the firmware
  (`asyncio`), then in `/lib`. It finds `mod.py`, or `mod.mpy` when there
  is no `mod.py`.

## Modules

`sys`, `os`, `time`, `json`, `re`, `math`, `cmath`, `random`,
`binascii`, `hashlib`, `heapq`, `collections`, `struct`, `array`, `errno`,
`gc`, `io`, `select`, `asyncio`, `micropython`, `machine` and `neopixel`
(see [Hardware](#hardware-machine)), `tdui` (see [Windows](#windows-tdui))
and `tinydesk`.
`help('modules')` lists them on the device.

Not included: `network`, `socket`, `_thread`, `bluetooth`,
`deflate`, `framebuf`, `cryptolib`, `vfs`. Numbers: integers of any size,
and single-precision floats (about 7 digits, the same on every board and
PC).

* `hashlib`: `sha256`, `sha1` and `md5`. On the boards they use mbedTLS
  and the chip's SHA hardware. The Windows and Linux programs have `sha1`
  and `md5` when they were built with mbedTLS, as the released ones are
  (the same condition as MQTT over TLS).
* `os.urandom(n)`: `n` random bytes from TinyDesk's random source (the
  chip's hardware generator on a board).
* `select.poll()` works on objects that support it; TinyDesk has no
  sockets, and whether a key is waiting on `sys.stdin` is not known, so
  its main use is `asyncio`'s loop.

`time.time()` and `time.localtime()` follow the clock TinyDesk keeps (SNTP
or `date`); `time.ticks_ms()` and `time.sleep()` work as in MicroPython.

## The tinydesk module

```python
import tinydesk
```

| Function | What it does |
|---|---|
| `tinydesk.version()` | TinyDesk's version, for example `'0.2.0'`. |
| `tinydesk.shell(line)` | Runs a shell command line in the script's session and returns its exit status. Its output goes where the script's output goes. It starts in the script's current directory; a `cd` in it does not move the shell. |
| `tinydesk.log(text, level='I')` | Adds a line to the Log Viewer. `level` is `'E'`, `'W'`, `'I'` or `'D'`. |
| `tinydesk.mqtt_publish(topic, payload, qos=0, retain=False)` | Publishes on the broker the MQTT app or `mqtt connect` connected to. `payload` is a `str` or `bytes`; `qos` is 0 or 1. Raises `OSError` when MQTT is not connected. |
| `tinydesk.modbus_read(target, unit, table, addr, count=1)` | Reads Modbus values and returns a list of ints. Arguments as for [`modbus read`](commands.md#modbus): `target` is `'host[:port]'` or `'rtu[N][:baud[:8N1]]'`, `table` is `'co'`, `'di'`, `'hr'` or `'ir'`, `count` 1..125. |
| `tinydesk.modbus_write(target, unit, table, addr, values)` | Writes one value (an int) or several (a list) to coils (`'co'`) or holding registers (`'hr'`), up to 64 values. |

Errors from MQTT and Modbus raise `OSError` with the same text the shell
commands print, for example `OSError: modbus: No connection (timeout)`.

```python
import json, time, tinydesk

# Read two holding registers every 10 s and publish them as JSON.
while True:
    hr = tinydesk.modbus_read("192.168.1.50", 1, "hr", 0, 2)
    tinydesk.mqtt_publish("plant/line1", json.dumps({"temp": hr[0] / 10, "rpm": hr[1]}))
    time.sleep(10)
```

## asyncio

`asyncio` runs several coroutines in one script: for example one that
reads a Modbus device every second, one that publishes on MQTT every
minute and one that writes a file every hour, each with its own timing.

```python
import asyncio, tinydesk

async def poll():
    while True:
        hr = tinydesk.modbus_read("192.168.1.50", 1, "hr", 0, 2)
        tinydesk.log("hr=%d,%d" % (hr[0], hr[1]))
        await asyncio.sleep(1)

async def report():
    while True:
        await asyncio.sleep(60)
        tinydesk.mqtt_publish("plant/alive", "1")

async def main():
    await asyncio.gather(poll(), report())

asyncio.run(main())
```

How it fits into TinyDesk:

* **One task.** The event loop runs inside the script, in the shell task
  the script runs in (the Terminal window's, or a Telnet or SSH
  session's). It switches only between the script's own coroutines: the
  desktop, Wi-Fi, MQTT, the Modbus server and other sessions keep running
  in their own FreeRTOS tasks (threads on a PC), as they do for any
  script.
* **Waiting sleeps.** When every coroutine is waiting (`asyncio.sleep()`,
  an `Event`, a `Lock`), the loop sleeps the shell task in steps of up to
  10 ms until the next one is due, so an idle script takes no CPU time.
  Ctrl+C stops it there as anywhere else (exit status 130).
* **The `tinydesk` calls block.** `modbus_read()`, `modbus_write()`,
  `mqtt_publish()` and `shell()` are ordinary functions: while one runs
  (a Modbus request up to its timeout, a shell command until it ends), no
  other coroutine runs. Keep them short, or run them less often.
* **No I/O to wait on.** Without sockets, and without a way to know that
  a key is waiting, `asyncio`'s streams have nothing to read in TinyDesk:
  coroutines wait for time, `Event`s and each other.

It is MicroPython's `asyncio` (the functions are listed in
[its documentation](https://docs.micropython.org/en/v1.29.0/library/asyncio.html)).
One difference from CPython: `asyncio.sleep()` is not a coroutine, so
`wait_for()` needs a coroutine around it.

## Precompiled .mpy files

`mpy-cross`, MicroPython's compiler, turns a `.py` file into a `.mpy`
file: bytecode that loads without being compiled on the device, which
needs less RAM (the compiler's work is done) and starts faster. Use an
`mpy-cross` from MicroPython 1.19 or later (`pip install mpy-cross`, or
the one `tools/micropython` builds); for files with machine code, one from
1.29:

```
mpy-cross sensors.py            makes sensors.mpy
```

Copy the `.mpy` file to the device (with FTP, `scp`, or the Files app);
`py sensors.mpy` runs it, and `import sensors` loads it from the script's
directory or `/lib`. `.mpy` files are the same on every board and PC,
unless they contain machine code (next section).

## Native code (root only)

`@micropython.native` compiles a function to machine code instead of
bytecode, and `@micropython.viper` to faster machine code with typed
integers and raw pointers (`ptr8`, `ptr16`, `ptr32`). `mpy-cross -march=`
does the same for `.mpy` files.

```python
import micropython

@micropython.viper
def checksum(buf) -> int:
    p = ptr8(buf)
    s = 0
    for i in range(int(len(buf))):
        s += p[i]
    return s & 0xFFFF
```

* **Only root.** Machine code can read and write all of the device's RAM,
  past TinyDesk's users and file permissions, so for any other user a
  native function, or loading a `.mpy` file with machine code, raises
  `RuntimeError: machine code (native, viper, native .mpy) needs root`.
* **Where.** The ESP32 and the ESP32-S3 (`mpy-cross -march=xtensawin`),
  the ESP32-C6 (`-march=rv32imc`) and the Linux program on x86-64
  (`-march=x64`). A `.mpy` file made for another chip is refused with
  `ValueError: incompatible .mpy arch`. For this, the ESP32-S3's and the
  ESP32-C6's firmware turn off the memory protection that keeps RAM
  non-executable, as MicroPython's own firmware does; the classic ESP32
  has none. Not the Windows program: MicroPython has no emitter for the
  Windows x64 calling convention. There, `@micropython.native` is a
  `SyntaxError` and a native `.mpy` file is refused.
* **Memory.** The code goes to executable RAM outside the Python heap and
  is freed when the script ends. On the classic ESP32 that is the IRAM the
  firmware leaves: up to about 10 KB on the ESP32 with PSRAM and about
  33 KB on the 4 MB ESP32, enough for a few small hot functions (each takes
  a few hundred bytes to a few KB). On the ESP32-S3 and the ESP32-C6 it
  comes from the internal RAM everything else uses. When it runs out,
  `MemoryError`.
* **Ctrl+C** does not stop a loop inside a native or viper function that
  calls nothing; it takes effect when the function returns.

## Hardware (`machine`)

On the ESP32 boards, root's scripts drive the chip's pins and buses with
MicroPython's usual `machine` API, so examples and sensor drivers written
for MicroPython work as they are:

```python
from machine import Pin, PWM, ADC, I2C
import time

led = Pin(25, Pin.OUT)
button = Pin(26, Pin.IN, Pin.PULL_UP)
button.irq(lambda p: print("pressed"), Pin.IRQ_FALLING)
for i in range(6):
    led.toggle()
    time.sleep_ms(250)

dim = PWM(Pin(27), freq=1000, duty_u16=16384)    # 25 %
print(ADC(Pin(32)).read_uv(), "microvolts")
i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=400000)
print(i2c.scan())                                 # addresses that answer
```

* **Only root**, as for the shell's `gpio` command: for any other user,
  using a pin or a bus raises `OSError: machine: hardware needs root`.
* **Pins TinyDesk uses are refused**, with what uses them:
  `OSError: machine: GPIO 16 is used by TinyDesk (rs485.1.tx)`. Those are
  the pins in the [board configuration](../guide/board-config.md)
  (`console.*`, `rs485.*`, `eth.*`, `sd.*`), the desktop's link (UART0 on
  GPIO1 and 3 on the classic ESP32 and GPIO43 and 44 on the ESP32-S3
  unless `console.*` says otherwise; USB on GPIO12 and 13 of the ESP32-C6),
  and the pins the chip keeps (flash, and PSRAM on the ESP32 with PSRAM and
  the ESP32-S3: GPIO26–37 there).
* **Strapping pins** (ESP32: 0, 2, 5, 12, 15; ESP32-S3: 0, 3, 45, 46;
  ESP32-C6: 8, 9, 15) work, with
  a warning the first time: what is wired to them can keep the board from
  starting.
* **Everything is undone when the script ends**, normally, after an error
  or after Ctrl+C: interrupts are removed, PWM outputs, buses and UARTs
  closed, and the pins the script changed reset to inputs.
* **The PC programs** have the module, without hardware: `import machine`
  works, and using a pin or a bus raises
  `OSError: machine: no hardware in this program`.

| Class or function | Notes |
|---|---|
| `Pin(n, mode, pull, *, value)` | GPIO numbers. `Pin.IN`, `Pin.OUT`, `Pin.OPEN_DRAIN`; `Pin.PULL_UP`, `Pin.PULL_DOWN`. `value()`, `on()`, `off()`, `toggle()`, `init()`; an output reads back its own level. `Pin(n)` without a mode leaves the pin as it is. Input-only pins (ESP32 GPIO34–39) refuse `OUT`. |
| `pin.irq(handler, trigger)` | `Pin.IRQ_RISING`, `Pin.IRQ_FALLING` or both (the default). The handler gets the `Pin`; it runs between the script's bytecodes, shortly after the edge (not inside the interrupt), so it may allocate and print. `irq(None)` removes it. |
| `ADC(pin)` | `read_u16()` (0–65535) and `read_uv()` (microvolts, calibrated where the chip has calibration data). 12 bits, input range about 0–3.1 V. ADC1 pins only: ESP32 GPIO32–39, ESP32-S3 GPIO1–10, ESP32-C6 GPIO0–6 (ADC2 is unusable while Wi-Fi is on). |
| `PWM(pin, freq, *, duty_u16, duty_ns)` | `freq()`, `duty_u16()`, `duty_ns()`, `deinit()`. 6 outputs on the ESP32-C6, 8 on the ESP32 and the ESP32-S3; up to 4 different frequencies at a time (outputs with the same frequency share a timer). Duty resolution up to 14 bits, less above about 5 kHz. Starts at duty 0. |
| `I2C(id, *, scl, sda, freq=400000, timeout=50000)` | Hardware I2C 0 (and 1 on the ESP32 and the ESP32-S3); the pins are given. `scan()`, `readfrom()`, `writeto()`, `readfrom_mem()`, `writeto_mem()` and the rest of MicroPython's I2C methods. Every transfer ends with a stop (`stop=False` is not kept); a register read is one write-then-read transfer. The chip's internal pull-ups are on; most buses still need real ones. |
| `SPI(id, baudrate, *, polarity, phase, firstbit, sck, mosi, miso)` | Hardware SPI 1 (and 2 on the ESP32 and the ESP32-S3), 8-bit words. A bus TinyDesk uses for Ethernet or the SD card is refused: on an ESP32-C6 with either, use `SoftSPI`. |
| `SoftI2C(scl, sda, ...)`, `SoftSPI(sck, mosi, miso, ...)` | Driven by the CPU on any free pins. |
| `UART(id, baudrate, bits, parity, stop, *, tx, rx, rxbuf, timeout, timeout_char)` | A UART that is neither the console nor an RS-485 line (`console.uart`, `rs485.N.uart`): UART1 or 2 on the ESP32 and the ESP32-S3 (UART0 is the console), UART0 or 1 on the ESP32-C6; none on an ESP32-C6 whose two RS-485 lines use both. `read()`, `readline()`, `write()`, `any()`, `txdone()`, `flush()`; works with `select` and asyncio streams. Waiting for data still lets Ctrl+C through. |
| `time_pulse_us(pin, level, timeout_us)` | Busy-waits; returns -2 or -1 on time-out. |
| `bitstream(pin, 0, timing, buf)`, module `neopixel` | WS2812-type LEDs through the RMT peripheral: `np = neopixel.NeoPixel(Pin(18), 8)`, `np[0] = (255, 0, 0)`, `np.write()`. |
| `unique_id()`, `freq()`, `idle()` | The factory MAC address; the CPU clock (setting it is refused). |

Not included: `Timer`, `RTC`, `WDT`, `deepsleep()`, `lightsleep()`,
`reset()`, `mem8`/`mem16`/`mem32`, `I2S`, `CAN`, `Signal`, touch pins and
the DAC. Sleep, reset, the clock and the watchdog belong to TinyDesk; a
script that must restart the board can run `tinydesk.shell("reboot")`.

Cost: about 65 KB of flash on the ESP32 boards and 75 KB on the ESP32-C6.
Internal RAM: about 7 KB less free at boot on the ESP32-C6, where the
drivers' interrupt code that ESP-IDF keeps in RAM shares memory with the
heap, and under 1 KB taken once by the drivers the first time a script
uses them. Nothing else stays allocated between scripts.

## Windows (`tdui`)

A script can put windows on the desktop, with labels, buttons, text boxes,
check boxes, lists, progress bars and character canvases, and handle what
the user does in Python. That makes a small app without writing C:

```python
import tdui

w = tdui.Window("Counter", 32, 9)        # client area: 32 x 9 cells, centred
count = w.label(2, 1, "0", width=10)
name = w.textbox(2, 3, 20)
loud = w.checkbox(2, 5, "Shout")

def add(button):
    count.text = str(int(count.text) + 1)

def greet(button):
    msg = "Hello " + name.text
    tdui.msgbox("Hi", msg.upper() if loud.checked else msg)

w.button(2, 7, "+1", add)
w.button(10, 7, "Greet", greet)
tdui.run()                               # until the window is closed
```

How it works:

* **The desktop draws, the script decides.** The window manager runs in
  the desktop's own task and the script in the shell's, so a script never
  touches a window directly: it sends requests (open this window, set this
  label) and the desktop carries them out within 10 ms. What the user does
  (a click, typing, closing the window) comes back as events, and
  `tdui.run()` or `tdui.poll()` calls the script's handlers. A slow handler
  holds up only the script, never the desktop.
* **Reading a widget does not wait.** The desktop sends every change of a
  text box, check box or list selection to the script as it happens, so
  `name.text` or `loud.checked` is read at once, and is up to date when a
  button's handler runs.
* **Where.** Only a script started from the desktop's own console has a
  desktop: the Terminal window, the Editor (F5), or Run on the desktop and in
  Files. In an SSH session `tdui.Window()` raises
  `OSError: tdui: no desktop in this session`. It works the same in the
  Windows and Linux programs.
* **Who:** any user; the windows appear on that user's desktop.
* **The windows belong to the script.** They close when it ends, also after
  an error or Ctrl+C. Ctrl+C in one of the script's windows stops it as it
  does in the Terminal. A script that does not call `tdui.run()` (or keep
  calling `poll()`) ends at once, and so do its windows.
* **Limits**, so a script cannot crowd out the desktop's own apps: 4
  windows, 48 widgets (a list counts as 2, for its scrollbar) and 2000
  canvas cells in all. Past them, `OSError` says which. Labels, buttons and
  text boxes hold up to 63 bytes, a window title 31.

| Class or function | Notes |
|---|---|
| `Window(title, w, h, x=None, y=None, resizable=False)` | `w` and `h` are the client area in cells; without `x`, `y` the window is centred. `.title`, `.focus()`, `.close()`, `.is_open`; `.on_close(win)` runs when the user closes it; `.on_key(win, key)` gets the keys no widget used, as `'a'`, `'enter'`, `'f5'`, `'ctrl+s'`, `'shift+tab'`. `.every(ms, fn)` and `.after(ms, fn)` run `fn()` repeatedly or once while the window is open; they return a timer with `cancel()`. See [Size and place](#size-and-place). |
| `w.label(x, y, text, width=0)` | `width` 0: to the right edge. |
| `w.button(x, y, text, on_click)` | `on_click(button)`: a click, Enter or Space. |
| `w.checkbox(x, y, text, checked=False, on_change=None)` | `.checked`; `on_change(box)`. |
| `w.textbox(x, y, width, text="", secret=False, on_enter=None, on_change=None)` | `.text`; `on_enter(box)` on Enter, `on_change(box)` after each edit. `secret` shows `*`. |
| `w.progress(x, y, width, value=0)` | `.value`, 0–100. |
| `w.list(x, y, w, h, items, on_select=None, on_activate=None)` | `.items` (a list of strings; set it to replace them), `.append(item)`, `.selected`; `on_select(lst)` when the selection moves, `on_activate(lst)` on Enter or a double-click. Up to 1000 items. `w` or `h` 0 or less: to that many cells before the window's right or bottom edge, following resizes. |
| `w.canvas(x, y, w, h, on_mouse=None)` | A grid of character cells: `put(x, y, text, fg=-1, bg=-1)`, `fill(x, y, w, h, ch=" ", fg=-1, bg=-1)`, `clear()`. Colours are the terminal's 256 palette numbers, -1 the window's own. `on_mouse(canvas, x, y, button, action)`: `action` 0 press, 1 release, 2 drag. `.width`, `.height`; `resize(w, h)` keeps what fits of the picture, `move(x, y)` places it elsewhere in the window. |
| Every widget | `.text`, `.visible`, `.color(fg, bg)`, `.focus()`. |
| `msgbox(title, text, buttons="OK")` | Buttons separated by `\|`, for example `"Yes\|No"`; returns the number of the one chosen (from 0), -1 for Esc. |
| `confirm(title, text)` | Yes / No; `True` for Yes. |
| `inputbox(title, prompt, text="", secret=False)` | The text entered, or `None` for Cancel. |
| `menu(items)` | A popup menu at the mouse; the number of the item chosen, -1 if none. An item `"-"` is a separator. Up to 20 items. |
| `run()` | Handles events and timers until every window is closed. |
| `poll(ms=0)` | Handles what happened and the timers that are due, for up to `ms` milliseconds: for a script with its own loop. |
| `run_async()` | `run()` for [asyncio](#asyncio): `await tdui.run_async()` while other tasks keep going. |
| `every(ms, fn)`, `after(ms, fn)`, `quit()` | Timers not tied to a window; `quit()` closes all the script's windows. |

### Size and place

| Method or attribute | Notes |
|---|---|
| `win.width`, `win.height` | The client area's size now. The desktop may give less than asked for on a small screen, and the user can resize a window made with `resizable=True`; these follow. |
| `win.on_resize(win, w, h)` | Runs after every change of size: the user dragging the corner, maximising, full screen (F11), and the script's own calls below. |
| `win.resize(w, h)`, `win.move(x, y)` | A new client size, or the window's top-left corner on the screen. A maximised window is restored first. Works without `resizable=True`. |
| `win.maximize()`, `win.minimize()`, `win.restore()`, `win.fullscreen()` | `restore()` also brings a minimised window back. Maximise needs `resizable=True`. |

Widgets placed with a negative `x` or `y` (from the right or bottom edge),
labels with `width=0` and lists with a size of 0 or less follow a resize by
themselves; canvases keep their size until the script calls
`canvas.resize()`, usually in `on_resize`:

```python
w = tdui.Window("Sketch", 40, 12, resizable=True)
cv = w.canvas(0, 0, 40, 11)
w.label(0, -1, "drag the corner to resize", width=0)

def grow(win, width, height):
    cv.resize(width, height - 1)       # within the 2000-cell limit

w.on_resize = grow
```

The dialogs wait for the answer, while the script's other windows and
timers keep working. Each script has its own windows: they are not in the
start menu, and two scripts cannot run at once (see
[Running scripts](#running-scripts)).

Cost: about 14 KB of flash. Nothing in RAM until a script opens a window;
then about 12 KB of internal RAM for the queues and the window table while
it runs, plus 6 bytes per canvas cell and the list items, all freed when it
ends.

## Memory

The interpreter's heap is allocated when `py` starts and freed when it
ends; nothing stays allocated between scripts. MicroPython itself keeps
under 1 KB of RAM.

| Device | Default heap | From |
|---|---|---|
| ESP32 with PSRAM, ESP32-S3 | 512 KB | PSRAM |
| ESP32-C6 | 64 KB | internal RAM |
| ESP32 4 MB (no PSRAM) | 64 KB, usually less (see below) | internal RAM |
| Windows, Linux | 1 MB | the PC's memory |

From internal RAM `py` always leaves 32 KB free for Wi-Fi, SSH and the
desktop, and takes less than the default when that is all there is. With
less than 16 KB to give it does not start, and prints how much RAM was
free. On the 4 MB ESP32 the desktop leaves about 65 KB free with Wi-Fi
off, so a script gets about 31 KB; with Wi-Fi connected there may not be
enough to start one. `heap` in the shell shows the free RAM;
`gc.mem_free()` shows what is left inside a script, and
`micropython.mem_info()` shows more.

A script's frames use the shell task's stack; deep recursion ends with
`RuntimeError: maximum recursion depth exceeded`, not a crash.

## For developers

The interpreter is in `third_party/micropython`, generated from the
MicroPython release by `tools/micropython` (see its README), including
`asyncio`'s Python part, compiled to bytecode and built into the firmware
(`frozen/`). TinyDesk's port of it, including the configuration
(`mpconfigport.h`), the frozen-module list (`manifest.py`), the file
system, the `tinydesk` module and the `machine` module, is in
`ports/common/python`; the chip side of `machine` (ESP-IDF's drivers, and
the rules for which pins a script may use) is in its `hw/` folder. `tdui`
is `modules/tdui.py` there, over the small C module `_tdui`; the desktop
side, which makes the windows, is `apps/pyui.c`, and the messages between
the two are in `apps/td_apps.h`. Builds can
leave it out with `-DTD_WITH_PYTHON=OFF` (PC). See [Tools](../tools.md).
