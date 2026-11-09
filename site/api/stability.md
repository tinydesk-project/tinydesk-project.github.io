# API stability

From 1.0, TinyDesk and TinyDesk Shell follow
[semantic versioning](https://semver.org): an app, script, `.tdapp`
package or board configuration made for 1.x keeps working on every later
1.x release.

| Release | May |
| --- | --- |
| patch (1.0.1) | fix bugs; no API change |
| minor (1.1.0) | add functions, types, struct members at the end, constants, widgets, events, Python names, board keys |
| major (2.0.0) | remove or change anything public |

To test for a version in C: `TD_VERSION_MAJOR`, `TD_VERSION_MINOR`,
`TD_VERSION_PATCH` (TinyDesk, `tinydesk/td.h`) and `TDSH_VERSION_MAJOR`,
... (the shell, `tdsh.h`). `TD_VERSION` and `TDSH_VERSION` are the same as
text.

## What is promised

| Part | What | Pages |
| --- | --- | --- |
| C API | the functions, types and constants in TinyDesk's `include/tinydesk/*.h`, `apps/td_apps.h`, `proto/td_*.h`, and the shell's `include/*.h`, `tdsh_espidf.h`, `td_py.h` | this API reference |
| Python | `tdui`, `tdapp`, `tinydesk`, `tdsh`, `network`, `socket`, `ssl`, `machine`, `neopixel`, `websocket`, `requests`, `umqtt`, `aiohttp` | [Python scripts](../shell/python.md), [tdapp](../shell/tdapp.md) |
| `.tdapp` packages | package format 1 | [tdapp](../shell/tdapp.md) |
| board.conf | every key, its meaning and its default | [Board configuration](../guide/board-config.md) |
| Commands | names, options, exit statuses (not the exact text they print) | [Commands](../shell/commands.md) |
| Your data | settings, accounts, Wi-Fi networks and files: a newer 1.x reads them | |

Not promised: functions the reference calls internal (the core's frame and
window-manager plumbing, in `src/td_internal.h`; the shell's command
handlers and start routines, in `tdsh_espidf_internal.h`), the look of the
desktop, and the exact text commands print. MicroPython's own modules
follow MicroPython.

## How it is kept

Each repository records its public API: `api/tinydesk.api` and
`api/tdsh.api` (every declaration of the public headers), the Python
contracts (`tdui_contract.json`, `tdapp_contract.json`, the shell's
`py_contract.json`) and `api/board_keys.txt`. The tests fail on any
difference, so a change to the API is always deliberate and visible in the
diff. Within 1.x a recorded change may only add. Something to be retired is
marked deprecated, keeps working, and goes in the next major version. The
details for contributors are in each repository's `STABILITY.md`.
