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
module. They do not drive hardware directly: TinyDesk owns the pins, the
network and the connections.

## Running scripts

Usage: `py [-m KB] [FILE [ARG...] | -c CODE [ARG...]]`

| Form | What it does |
|---|---|
| `py FILE [ARG...]` | Runs the script: Python source, or a `.mpy` file (see [Precompiled .mpy files](#precompiled-mpy-files)). `sys.argv` is `[FILE, ARG...]`; the script's directory comes first in `sys.path`, so it can import modules next to it. |
| `py -c CODE [ARG...]` | Runs `CODE`; `sys.argv` is `['-c', ARG...]`. |
| `py` | The interactive prompt, with history, auto-indent and paste mode (Ctrl+E, then Ctrl+D to run). Ctrl+D or `exit()` leaves it. `help()` lists what is available. |
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
`gc`, `io`, `select`, `asyncio`, `micropython`, and `tinydesk`.
`help('modules')` lists them on the device.

Not included: `machine`, `network`, `socket`, `_thread`, `bluetooth`,
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
| `tinydesk.version()` | TinyDesk's version, for example `'0.1.6'`. |
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
* **Where.** The ESP32 boards (`mpy-cross -march=xtensawin`) and the Linux
  program on x86-64 (`-march=x64`). Not the ESP32-C6: its firmware keeps
  RAM non-executable, a protection MicroPython's own firmware turns off and
  TinyDesk keeps. Not the Windows program: MicroPython has no emitter for
  the Windows x64 calling convention. There, `@micropython.native` is a
  `SyntaxError` and a native `.mpy` file is refused.
* **Memory.** The code goes to executable RAM outside the Python heap and
  is freed when the script ends. On the ESP32 that is the IRAM the firmware
  leaves: up to about 10 KB on the ESP32 with PSRAM and about 33 KB on the
  4 MB ESP32, enough for a few small hot functions (each takes a few
  hundred bytes to a few KB). When it runs out, `MemoryError`.
* **Ctrl+C** does not stop a loop inside a native or viper function that
  calls nothing; it takes effect when the function returns.

## Memory

The interpreter's heap is allocated when `py` starts and freed when it
ends; nothing stays allocated between scripts. MicroPython itself keeps
under 1 KB of RAM.

| Device | Default heap | From |
|---|---|---|
| ESP32 with PSRAM | 512 KB | PSRAM |
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
system and the `tinydesk` module, is in `ports/common/python`. Builds can
leave it out with `-DTD_WITH_PYTHON=OFF` (PC). See [Tools](../tools.md).
