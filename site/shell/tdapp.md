# Python apps library (tdapp)

`tdapp` is the library TinyDesk apps are built on. [`tdui`](python.md#windows-tdui)
gives an app its windows, widgets and canvases; `tdapp` adds what nearly
every app needs on top of them, so the app's own file holds only its own
logic: its widgets, its artwork and what happens on a click.

| Part | What it does |
| --- | --- |
| [`App`, `Page`](#app-and-pages) | a window with pages: show one, the others' widgets and canvases hide |
| [`Link`](#serial-links-link) | Modbus RTU master or slave, or a plain UART protocol, on a serial port |
| [Bindings](#bindings) | a widget shows (or writes) a register without a callback |
| [`Field`, `gauge`, `slider`, `panel`](#canvas-elements) | canvas elements that redraw themselves when a value changes |
| [Canvas pages](#canvas-pages) | a canvas with a different picture per window page (what App Builder's apps use) |
| [`require`, `batch`](#firmware-features-and-drawing) | a clear message on old firmware; several drawings as one frame |

It is frozen into the firmware next to `tdui`, on every board with Python:
`import tdapp`. App Builder's apps use it too, so an app written by hand and
one made in App Builder share the same, tested code.

## A whole app

```python
import tdapp

app = tdapp.App("Oven", 44, 14)
win = app.win

main = app.page("Main")
settings = app.page("Settings")

main.add(win.label(2, 1, "Temperature"))
temperature = main.add(win.label(16, 1, "--"))
cv = main.canvas(2, 3, 24, 8)
dial = tdapp.Field(cv, 0, "gauge", {"x": 0, "y": 0, "w": 48, "h": 16, "value": 0, "maximum": 400})

settings.add(win.label(2, 1, "Set point"))
setpoint = settings.add(win.textbox(14, 1, 8, "250"))

win.button(30, 3, "Main", lambda b: app.show("Main"))
win.button(30, 5, "Settings", lambda b: app.show("Settings"))

link = app.link({"mode": "modbus_slave", "uart": 1, "tx": 17, "rx": 18, "baud": 9600, "unit": 95,
                 "registerCount": 8}, simulated=True)
app.bind(temperature, {"table": "hr", "address": 0, "suffix": " C"})

def measured():
    t = read_thermocouple()          # the app's own logic
    link.put("hr", 0, t)             # for the master to read; the label follows
    dial.set(value=t)

app.every(500, measured)
app.show("Main")
app.run()
```

The same program is `examples/python/tdapp_demo.py` in the sources (with a
made-up temperature). `simulated=True` opens no UART: drop it on the board.

## App and pages

```python
app = tdapp.App(title, w, h, x=None, y=None, resizable=False, fullscreen=False)
app.win                      # the tdui.Window
page = app.page(name)        # made the first time
page.add(widget, ...)        # returns the widget (or a tuple)
cv = page.canvas(x, y, w, h, on_mouse=None, pages=1)
app.show(name)               # ValueError for a page that does not exist
app.current                  # the page shown, or None
app.on_page = fn             # fn(name) after each show()
app.every(ms, fn), app.after(ms, fn)
app.link(config, simulated=False)   # a Link, ticked every 10 ms
app.bind(widget, spec, kind=None)
app.run()                    # until the windows close; then closes the link
```

Widgets that belong to no page (the two buttons above) are always shown.
A page's labels, buttons, check boxes, text boxes, progress bars and lists
are hidden with their `visible` property. `tdui` canvases cannot be hidden
that way, so a page moves its canvases off the window and back: make them
with `page.canvas()`, not `win.canvas()` (`page.add()` refuses a canvas it
did not make). `show()` draws the change as one frame.

## Serial links (Link)

```python
link = tdapp.Link(config, simulated=False)   # or app.link(config)
link.start()
win.every(10, link.tick)                     # app.link() does both
link.put(table, address, *words)             # set words in the own bank and redraw bindings
link.get(table, address, count=1)            # [words]
link.write(spec, value)                      # a bound control's write, by hand
link.bind(widget, spec, kind)
link.on_write = fn                           # fn(table, address, words) after a remote write
link.before_read = fn                        # fn(table, address, count) before a reply
link.status_target = widget                  # shows WAIT, ONLINE, TIMEOUT, STALE, ERROR ...
link.state, link.last_error, link.frames, link.errors, link.rx_bytes, link.tx_bytes
link.close()
tdapp.crc16(data)                            # Modbus CRC
```

`config` is a dict; the keys left out take these values:

| Key | Default | Meaning |
| --- | --- | --- |
| `mode` | | `modbus_master`, `modbus_slave`, `uart_master` or `uart_slave` |
| `uart`, `tx`, `rx` | | the UART (0-2) and its pins |
| `baud`, `bits`, `parity`, `stop` | 9600, 8, `none`, 1 | line settings (Modbus RTU needs 8 bits) |
| `de`, `deActive` | -1, 1 | an RS-485 driver-enable pin and its active level (-1: none) |
| `unit` | 1 | the remote station (master) or the app's own (slave), 1-247 |
| `pollMs`, `timeoutMs`, `retries`, `staleMs` | 250, 300, 1, 2000 | master polling; how long until values count as stale |
| `codec`, `codecModule`, `framing` | `rtu`, `""`, `newline` | UART modes: `rtu`, `json` (JSON Lines) or `custom` (a module of the app's own) |
| `registerCount`, `inputAlias` | 0, False | slave: holding registers 0 .. count-1; FC04 reads them too |

A master polls the registers its bindings read and queues their writes; a
slave answers from its own bank (`put()` sets it). Serial ports need root;
the console's and the RS-485 lines' UARTs are refused. Tables are `hr`
(holding registers), `ir` (input registers), `co` (coils) and `di`
(discrete inputs); addresses are zero-based (a PLC's 40001 is `hr` 0). The
details, as App Builder shows them, are in its *Communications* guide.

## Bindings

`app.bind(widget, spec)` (or `link.bind(widget, spec, kind)`, kind being
the widget's type: `label`, `textbox`, `progress`, `checkbox`, `button`, or a
canvas element's: `gauge`, `slider`, `bar`, `picture`, `font_text`, ...)
shows a register with the widget, or writes it from the widget:

| Key | Default | Meaning |
| --- | --- | --- |
| `direction` | `read` | `read`, `write` or `readwrite` |
| `table`, `address` | `hr`, 0 | which register |
| `dataType`, `wordOrder` | `u16`, `high_first` | `u16`, `i16`, `u32`, `i32`, `f32`; 32-bit word order |
| `gain`, `offset`, `decimals` | 1, 0, 0 | shown = raw x gain + offset, with that many decimals |
| `prefix`, `suffix` | `""` | around the text |
| `min`, `max`, `initial` | 0, 65535, 0 | limits of writes; the value before the first reply |
| `event`, `action` | `click`, `value` | when a control writes (`click`, `press`, `release`, `change`, `enter`) and what (`value`, `constant`, `toggle`, `momentary`) |
| `writeValue`, `releaseValue` | 1, 0 | the constants written |

Stale or missing values show `--` on text and keep a gauge where it was.

## Canvas elements

```python
f = tdapp.Field(cv, page, kind, values, image=None, on_change=None, font=None)
f.set(value=42)     # redraws only when something changed, as one frame
f.draw()
tdapp.gauge(cv, x, y, w, h, value, maximum=100, color=220, track=244, bg=17, gradient=None, radius=0)
tdapp.slider(cv, x, y, w, h, value, maximum=100, color=220, track=17, knob=231, gradient=None, radius=0)
tdapp.panel(cv, img, x, y, w, h, src=None, border=16, text="", fg=231, scale=0, font=None)
tdapp.control_mouse(cv, x, y, button, action)   # from a canvas's on_mouse: drags sliders
```

A `Field` is one element on canvas page `page` (0 for a canvas with one
page) drawn by `kind`: `gauge`, `slider`, `panel`, or any `tdui.Canvas`
drawing method (`bar`, `picture`, `font_text`, `draw_text`, ...) with
`values` as its arguments (pixel coordinates for the pixel methods). A
slider field calls `on_change(field)` when dragged (pass the canvas's
mouse events to `control_mouse`). `panel` draws a picture as a nine-slice
frame of separate pictures, so a label inside it does not redraw the whole
frame.

## Canvas pages

What App Builder's apps use for canvases that look different on each page
of a window: one canvas with a page per window page where the firmware has
canvas pages (`tdapp.PAGED`), separate canvases moved off the window
(`tdapp.PARK`) where not.

```python
cv = tdapp.make_canvas(win, x, y, w, h, pages)
cv2 = tdapp.alias_canvas(cv, win, x, y, w, h)
tdapp.register_canvas(win, group, base, cv, own, page, x, y, paint, visible, mouse)
tdapp.show_canvases(win, name)      # each group: the canvas of window page name
cv = tdapp.canvas_page(cv, name)    # draw on window page name's canvas
tdapp.hide_canvas(cv, clear=False), tdapp.place_canvas(cv, x, y), tdapp.select_canvas(cv, page)
```

`register_canvas` records that canvas `cv` (canvas page `page`) of group
`group` belongs to window page `own` (None: every page), painted by
`paint()`, with `mouse` as its `on_mouse`. `App.show()` calls
`show_canvases()` for its window too.

## Firmware features and drawing

```python
tdapp.require(cells=0, pixels=0, rgb=False, cache=None, pictures=0, visible_pictures=0,
              fonts=False, gradients=False, pixel_widgets=False)
with tdapp.batch():
    ...                 # all of it appears in one frame
tdapp.preparing(True)   # batch() draws at once while pages are prepared hidden
tdapp.VERSION           # (1, 0)
```

`require()` raises `OSError` naming what to update when the board's
firmware lacks something the app uses (`rgb=True` also turns RGB565
pictures on, and `cache` sets the render cache in KiB). `batch()` is
`tdui.batch()` where the firmware has it, nothing otherwise.

## Versions

`tdapp.VERSION` is (major, minor). Within a major version names are only
added, never removed or changed, so an app written for 1.0 runs on every
1.x: `tools/check_tdui_contract.py` checks the library against
`ports/common/python/tdapp_contract.json` as it checks `tdui`, and
`tools/check_tdapp.py` runs it in the simulator. Check for a newer
feature with `tdapp.VERSION >= (1, 1)` or `hasattr()`.
