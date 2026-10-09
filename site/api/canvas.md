# Pixel canvas

A canvas is a surface of pixels that an app draws into and shows in its window. Every cell of the terminal shows two pixels, one above the other: the character U+2580 (`▀`, upper half block) with the upper pixel as its foreground colour and the lower pixel as its background colour. A canvas of `w` x `h` pixels therefore takes `w` columns and `(h + 1) / 2` rows. No terminal feature beyond 256 colours is needed, so it works in PuTTY, Windows Terminal, xterm and the [web terminal](../console/index.html), and the [diff renderer](screen.md) sends only the cells that changed: a still picture costs nothing after it has been drawn once.

Header: `include/tinydesk/td_canvas.h` (included by `td.h`)
Source: `src/canvas.c`
Example app: `apps/pixels.c` ([Pixels](#the-pixels-demo))

All functions must be called from the UI task, like the rest of the drawing API.

## Pixels and colours

```c
typedef struct
{
    int w, h;
    uint8_t *px; /* w * h palette indexes, row by row */
} td_canvas_t;
```

Each pixel is one byte: an index into the xterm 256-colour palette, the same colours as everywhere else in TinyDesk (see [Colours](screen.md#colours-and-attributes)). An app may write `px[y * w + x]` directly, or use the drawing calls below. `td_rgb256()` finds the nearest palette colour for an RGB value.

| Size | Bytes | Cells |
| --- | --- | --- |
| 42 x 30 (Pixels' default window) | 1,260 | 42 x 15 |
| 80 x 48 (an 80x25 screen, full screen) | 3,840 | 80 x 24 |
| 160 x 98 (a 160x50 screen, full screen) | 15,680 | 160 x 49 |

## Functions

```c
td_canvas_t *td_canvas_new(int w, int h);
void td_canvas_free(td_canvas_t *c);
bool td_canvas_resize(td_canvas_t *c, int w, int h);

void td_canvas_clear(td_canvas_t *c, uint8_t color);
void td_canvas_pixel(td_canvas_t *c, int x, int y, uint8_t color);
uint8_t td_canvas_get(const td_canvas_t *c, int x, int y);
void td_canvas_fill_rect(td_canvas_t *c, int x, int y, int w, int h, uint8_t color);
void td_canvas_line(td_canvas_t *c, int x0, int y0, int x1, int y1, uint8_t color);
void td_canvas_blit(td_canvas_t *c, int x, int y, const uint8_t *src, int w, int h, int stride);

void td_canvas_draw(const td_canvas_t *c, int x, int y);
uint8_t td_rgb256(uint8_t r, uint8_t g, uint8_t b);

typedef void *(*td_canvas_alloc_fn)(size_t bytes);
typedef void (*td_canvas_free_fn)(void *ptr);
void td_canvas_set_allocator(td_canvas_alloc_fn alloc_fn, td_canvas_free_fn free_fn);
```

| Function | What it does |
| --- | --- |
| `td_canvas_new` | A canvas of `w` x `h` pixels, all colour 0 (black). NULL without memory or for a size below 1 x 1. |
| `td_canvas_free` | Frees a canvas; NULL is ignored. |
| `td_canvas_resize` | Changes the size; every pixel becomes colour 0. Returns false and leaves the canvas as it was when there is not enough memory. |
| `td_canvas_clear` | Sets every pixel. |
| `td_canvas_pixel`, `td_canvas_get` | Set or read one pixel. Outside the canvas: nothing happens, or 0 is returned. |
| `td_canvas_fill_rect` | Fills a rectangle, clipped to the canvas. |
| `td_canvas_line` | A straight line, both ends included, clipped point by point (Bresenham). |
| `td_canvas_blit` | Copies `w` x `h` palette indexes (rows `stride` bytes apart) to `(x, y)`, clipped: a decoded image, a sprite, another program's frame buffer. |
| `td_canvas_draw` | Draws the canvas into the current drawing target with its top-left cell at `(x, y)`, relative to the drawing origin and clipped like `td_putc()`. With an odd height the last row's lower half is colour 0. Call it from `on_draw`. |
| `td_rgb256` | The nearest colour of the 6 x 6 x 6 cube (16-231) or the grey ramp (232-255), whichever is closer. |
| `td_canvas_set_allocator` | Where pixels are allocated (below). NULL for both restores `malloc()` and `free()`. Change it only while no canvas exists. |

## Memory

A canvas takes `w * h` bytes for its pixels plus a small header, allocated by `td_canvas_new()` and `td_canvas_resize()`. Like everything else in an app, create it when the window opens (or in the first `on_draw`, when the size is known) and free it in `on_close`.

The pixels come from the allocator set with `td_canvas_set_allocator()`, `malloc()` by default. The ESP-IDF ports set one at start-up on boards with PSRAM (`CONFIG_SPIRAM`): every canvas then goes to PSRAM with `heap_caps_malloc(..., MALLOC_CAP_SPIRAM)`, falling back to internal RAM only when PSRAM is full, and each allocation is logged in the Log Viewer (`canvas: 15680 bytes in PSRAM`). Boards without PSRAM (the ESP32-C6, the 4 MB ESP32) keep `malloc()`. The allocator receives the size in bytes and returns uncleared memory or NULL; the canvas clears it.

## Example

```c
static td_canvas_t *s_canvas;

static void on_draw(td_window_t *win, int cw, int ch)
{
    (void)win;
    if (!s_canvas)
        s_canvas = td_canvas_new(cw, ch * 2);        /* fill the client area */
    else if (s_canvas->w != cw || s_canvas->h != ch * 2)
        td_canvas_resize(s_canvas, cw, ch * 2);      /* the window was resized */
    if (!s_canvas)
        return;
    td_canvas_clear(s_canvas, td_rgb256(0, 0, 64));
    td_canvas_line(s_canvas, 0, 0, s_canvas->w - 1, s_canvas->h - 1, td_rgb256(255, 200, 0));
    td_canvas_draw(s_canvas, 0, 0);
}

static void on_close(td_window_t *win)
{
    (void)win;
    td_canvas_free(s_canvas);
    s_canvas = NULL;
}
```

For animation, change the pixels in `on_tick` (or draw them in `on_draw` from a time value advanced in `on_tick`) and call `td_win_invalidate()`.

## How fast it is

The terminal link is the limit, not the CPU: a cell whose two colours both changed costs about 23 bytes (the colour sequence and the 3 UTF-8 bytes of `▀`). The main loop paces frames to what the terminal shows ([Frame pacing](core.md#frame-pacing)), so a slow terminal gets fewer pictures instead of a growing backlog. Measured with Pixels' plasma (every pixel changes in every picture):

| Board and link | Default window (42 x 30) | Full screen |
| --- | --- | --- |
| ESP32-S3, built-in USB | 32 pictures/s (Pixels' own limit) | 24/s at 80x25 (564 KB/s); 6/s at 160x50 (633 KB/s) |
| ESP32-C6, built-in USB | 30/s | 18/s at 80x25 (425 KB/s) |
| ESP32 with PSRAM, UART at 921600 | 9/s | 1/s at 160x50 (87 KB/s) |
| PuTTY on the ESP32-C6 (what PuTTY can draw) | about 10/s (86 KB/s) | - |

## The Pixels demo

`apps/pixels.c` shows an animated plasma and the 256-colour palette in a resizable window, with the pictures per second, the link's KB/s and the canvas size on its last line. Space switches between plasma and palette, P pauses. It starts animated every time it opens, and its canvas (allocated while the window is open) follows the window's size: maximise it or press F11 for more pixels. It computes the plasma without floating point, with integer sine and colour tables made when it opens.
