# Pixel canvas

A canvas is a surface of pixels that an app draws into and shows in its window. Normally every cell of the terminal shows two pixels, one above the other: the character U+2580 (`▀`, upper half block) with the upper pixel as its foreground colour and the lower pixel as its background colour. A canvas of `w` x `h` pixels therefore takes `w` columns and `(h + 1) / 2` rows. No terminal feature beyond 256 colours is needed, so it works in PuTTY, Windows Terminal, xterm and the [web terminal](../console/index.html), and the [diff renderer](screen.md) sends only the cells that changed: a still picture costs nothing after it has been drawn once. With the user's *Fine* [picture detail](#picture-detail) a cell shows four pixels instead.

Header: `include/tinydesk/td_canvas.h` (included by `td.h`)
Source: `src/canvas.c`
Example apps: `apps/pixels.c` ([Pixels](#the-pixels-demo)) and `apps/imageview.c` (the Image Viewer, which shows BMP and PPM pictures: [apps.md](apps.md#pictures-td_image_))

All functions must be called from the UI task, like the rest of the drawing API.

## Pixels and colours

```c
typedef struct
{
    int w, h;
    uint8_t *px; /* w * h palette indexes, row by row */
} td_canvas_t;
```

Each pixel is one byte: an index into the xterm 256-colour palette, the same colours as everywhere else in TinyDesk (see [Colours](screen.md#colours-and-attributes)). An app may write `px[y * w + x]` directly, or use the drawing calls below. `td_rgb256()` finds the nearest palette colour for an RGB value, and `td_rgb256_dither()` a dithered one for pictures.

| Size | Bytes | Cells (Normal detail) |
| --- | --- | --- |
| 42 x 30 (Pixels' default window) | 1,260 | 42 x 15 |
| 80 x 48 (an 80x25 screen, full screen) | 3,840 | 80 x 24 |
| 160 x 98 (a 160x50 screen, full screen) | 15,680 | 160 x 49 |

## Picture detail

How many pixels a cell shows is each user's choice, in **Settings → Sizes → Pictures**:

| Detail | Pixels per cell | Characters | Canvas for `cols` x `rows` cells |
| --- | --- | --- | --- |
| Normal (`TD_CANVAS_HALF`, the default) | 1 across, 2 down | `▀` | `cols` x `2 * rows` |
| Fine (`TD_CANVAS_QUAD`) | 2 across, 2 down | the quadrants `▗ ▖ ▄ ▝ ▐ ▞ ▟`, a space | `2 * cols` x `2 * rows` |

Fine doubles the detail across, not down. A cell still has only two colours, so where its four pixels have three or four colours it shows the two that change the cell least (each pixel takes the nearer one): fine lines and edges get sharper, while dense colour detail can look patchy. A Fine pixel is half as wide as it is tall, so an app that draws shapes or pictures should stretch them across by `canvas width / columns` (the Image Viewer does). Fine needs a terminal font with the quadrant characters: Cascadia Mono or Cascadia Code (Windows Terminal's font, also selectable in PuTTY under Window → Appearance), DejaVu Sans Mono. PuTTY's default Courier New, and Consolas and Lucida Console, have `▄` and `▐` but not the quadrants, which then show as small boxes along edges. In ASCII mode canvases draw as Normal.

An app does not choose the detail; it sizes its canvas for the cells it covers with `td_canvas_fit()` and checks in every `on_draw` that the canvas still fits, because the user can change the detail at any time (every window is then drawn again).

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

typedef enum { TD_CANVAS_HALF, TD_CANVAS_QUAD } td_canvas_detail_t;
void td_canvas_set_detail(td_canvas_detail_t detail);
td_canvas_detail_t td_canvas_detail(void);
void td_canvas_fit(int cols, int rows, int *w, int *h);
void td_canvas_quad_cell(const uint8_t px[4], uint32_t *ch, uint8_t *fg, uint8_t *bg);

uint8_t td_rgb256(uint8_t r, uint8_t g, uint8_t b);
uint8_t td_rgb256_dither(uint8_t r, uint8_t g, uint8_t b, int x, int y);

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
| `td_canvas_draw` | Draws the canvas into the current drawing target with its top-left cell at `(x, y)`, relative to the drawing origin and clipped like `td_putc()`, in the current [picture detail](#picture-detail). Pixels a cell has no canvas pixel for (an odd height, or an odd width in Fine detail) are colour 0. Call it from `on_draw`. |
| `td_canvas_set_detail`, `td_canvas_detail` | Set or read the picture detail. The Settings app sets it from the user's settings; apps only read it. |
| `td_canvas_fit` | The canvas size in pixels that fills `cols` x `rows` cells in the detail drawn now: `cols` (Normal, or Fine in ASCII mode) or `2 * cols` (Fine) across, `2 * rows` down. |
| `td_canvas_quad_cell` | The character and the two colours that Fine detail draws for four pixels (upper left, upper right, lower left, lower right). |
| `td_rgb256` | The nearest colour of the 6 x 6 x 6 cube (16-231) or the grey ramp (232-255), whichever is closer. |
| `td_rgb256_dither` | The colour for the pixel at `(x, y)` with ordered (4 x 4 Bayer) dithering: each channel becomes one of the two palette levels around it, chosen by the pixel's position, so a patch of pixels averages to the RGB colour (within about 6 levels in the cube, 2 in greys). Palette colours stay exact (no speckles on white or black); colours whose channels are within 12 of each other use all 30 greys of the palette. For pictures and gradients; the Image Viewer uses it. |
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
    int w, h;
    td_canvas_fit(cw, ch, &w, &h);                  /* fill the client area */
    if (!s_canvas)
        s_canvas = td_canvas_new(w, h);
    else if (s_canvas->w != w || s_canvas->h != h)
        td_canvas_resize(s_canvas, w, h);           /* resized, or the detail changed */
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

Fine detail costs a little more: on the ESP32-S3 at 120x40, the default window 29 pictures/s (84 x 30 pixels) instead of 32, full screen 7/s (240 x 78 pixels, 501 KB/s) instead of 10/s (120 x 78, 596 KB/s).

## The Pixels demo

`apps/pixels.c` shows an animated plasma and the 256-colour palette in a resizable window, with the pictures per second, the link's KB/s and the canvas size on its last line. Space switches between plasma and palette, P pauses. It starts animated every time it opens, and its canvas (allocated while the window is open) follows the window's size: maximise it or press F11 for more pixels. It computes the plasma without floating point, with integer sine and colour tables made when it opens.
