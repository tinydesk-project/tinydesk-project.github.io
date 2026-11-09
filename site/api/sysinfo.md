# Platform services (sysinfo)

`td_sysinfo_t` is how a port tells TinyDesk about the platform and hands it optional services: memory and task figures, the system clock and time zone, a filesystem, network control, firmware updates, user accounts and settings storage. The apps only reach the platform through this structure, so they build unchanged on the ESP32 and on the desktop hosts. Every member may be left `NULL` (or 0); the apps then show "n/a", hide a control or show a message.

Header: `include/tinydesk/td_sysinfo.h` (included by `tinydesk/td.h`)
Sources: `src/td.c` (storage); implementations in `ports/esp_idf/app/main.c`, `net_esp.c`, `ota_esp.c` (shared by the ESP-IDF projects `ports/esp32c6`, `ports/esp32`, `ports/esp32-4mb` and `ports/esp32s3`), `ports/common/host_main.c` and `ports/common/td_fs_stdio.c`

## Threading

The apps call every callback from the UI task, often from `on_draw` or `on_tick`, so every callback must return quickly. Anything slow (a Wi-Fi scan, a firmware download, an SNTP sync) must be started in the background and report through a status call; the ESP32 port runs such jobs in short-lived FreeRTOS tasks. Data shared between such a worker and the callbacks must be protected by the port (`net_esp.c` and `ota_esp.c` use a mutex).

## Installing

```c
void td_set_sysinfo(const td_sysinfo_t *info);
const td_sysinfo_t *td_sysinfo(void);
```

`td_set_sysinfo()` stores the pointer, not a copy: the structure (and the ops tables it points to) must stay valid for the life of the program, so make it `static`. `NULL` resets to an empty structure. Call it before `td_init()` and before registering the apps, since `td_apps_register_all()` already reads the filesystem and settings.

`td_sysinfo()` returns the installed structure, or an empty one (all members `NULL`) when none was installed. It never returns `NULL`, so `td_sysinfo()->fs` is always safe to read; the members themselves must be checked.

```c
static td_sysinfo_t s_info;           /* static: TinyDesk keeps the pointer */

static uint32_t my_free_heap(void) { return board_free_ram(); }

void port_setup(const td_hal_t *hal)
{
    s_info.platform = "My board";
    s_info.chip = "XYZ-1, 1 core";
    s_info.free_heap = my_free_heap;
    s_info.fs = td_fs_stdio("/data");   /* see td_fs_stdio below */
    td_set_sysinfo(&s_info);
    td_init(hal);
    td_apps_register_all();
}
```

## td_sysinfo_t

```c
typedef struct {
    const char *platform;      /* "ESP32-C6", "Windows host", ... */
    const char *chip;          /* e.g. "ESP32-C6 rev 0.1, 1 core" */
    const char *sdk_version;   /* e.g. ESP-IDF version */

    uint32_t (*free_heap)(void);
    uint32_t (*min_free_heap)(void);
    uint32_t (*total_heap)(void);
    uint32_t (*psram_free)(void);
    uint32_t (*psram_total)(void);
    int (*task_count)(void);
    int (*cpu_mhz)(void);
    bool (*chip_temp)(int *tenths_c);
    int (*tasks)(td_task_info_t *out, int max);

    bool (*time_now)(int64_t *utc);
    bool (*time_set)(int64_t utc);
    bool (*time_auto_get)(void);
    bool (*time_auto_set)(bool on);
    bool (*time_sync)(void);

    bool (*get_tz)(long *seconds);
    bool (*set_tz)(long seconds);

    const td_net_ops_t *net;
    const td_display_ops_t *displays;
    const td_ota_ops_t *ota;   /* NULL: no updates on this platform */
    const char *no_ota_text;   /* with ota NULL: why, and how to update instead */
    const td_fetch_ops_t *fetch;   /* downloads (the Store), NULL: none */
    const char *store_url;     /* the Store's catalog, NULL: the official one */

    bool (*user_exists)(const char *user);
    bool (*authenticate)(const char *user, const char *password);
    const td_user_ops_t *users;        /* NULL: no roles, rights or Sign in list */

    bool (*settings_load)(void *data, int len);
    bool (*settings_save)(const void *data, int len);

    const td_fs_ops_t *fs;

    const char *extra;
    const td_desktop_file_t *desktop_files;

    const char *kiosk_app;              /* NULL or "": no kiosk mode */
    const char *kiosk_splash;           /* picture while the app starts, or NULL */
    bool kiosk_hide_status;             /* no status line behind the app */

    const td_mouse_ops_t *mouse;        /* NULL: no Mouse app */
    const td_keyboard_ops_t *keyboard;  /* NULL: no Keyboard app */
} td_sysinfo_t;
```

### Who implements what

| Member | ESP32-C6 / ESP32 (`ports/esp_idf/app/main.c`) | Windows / Linux / macOS host (`ports/common/host_main.c`) |
|---|---|---|
| `platform` | board name (`board_platform()`) | `"Windows host"`, `"Linux host"`, `"macOS host"` |
| `chip` | model, revision and cores from `esp_chip_info()` | `"host CPU"` |
| `sdk_version` | `"ESP-IDF v5.3.1"` style, from `esp_get_idf_version()` | `"n/a"` |
| `free_heap`, `min_free_heap`, `total_heap` | internal 8-bit RAM (`heap_caps_*`) | not set |
| `psram_free`, `psram_total` | `MALLOC_CAP_SPIRAM` (0 on the C6, which has none) | not set |
| `task_count` | `uxTaskGetNumberOfTasks()` | not set |
| `cpu_mhz` | `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ` | not set |
| `chip_temp` | the built-in temperature sensor (`temperature_sensor_*`), on the ESP32-S3 and ESP32-C6; not set on the classic ESP32, which has none | not set |
| `tasks` | `uxTaskGetSystemState()`, only when FreeRTOS trace facility and run-time stats are enabled | not set |
| `time_now` | `time()`, false before it is set | `time()` |
| `time_set` | `settimeofday()` | not set |
| `time_auto_get`, `time_auto_set`, `time_sync` | TinyDesk Shell's SNTP settings; sync in a short-lived task | not set |
| `get_tz`, `set_tz` | `~/.tdsh_tz` (TinyDesk Shell's `tz` file) | `~/.tdsh_tz`, else the PC's offset |
| `net` | `net_esp_ops()` (`net_esp.c`) | only through `td_host_use_net()` (the `tools/tdsim.c` simulator uses a fake one) |
| `displays` | `hal_mux_displays()` (`hal_mux.c`) | none ([Displays](#displays)) |
| `ota` | `ota_esp_ops()` (`ota_esp.c`) | not set |
| `fetch` | `fetch_esp_ops()` (`fetch_esp.c`) | `host_fetch_ops()` (`ports/common/host_fetch.c`: files copied, URLs through `curl`) |
| `store_url` | board key `store.url` (a shell path becomes a real one) | environment variable `TINYDESK_STORE` |
| `user_exists`, `authenticate` | TinyDesk Shell's user database | only through `td_host_use_users()` (tdsim test accounts) |
| `users` | TinyDesk Shell's accounts, roles and remembered login (`main.c`) | not set |
| `settings_load`, `settings_save` | NVS namespace `"tinydesk"`, key `"settings"` | file `tinydesk_settings.bin` in the working directory |
| `fs` | `td_fs_stdio("/fs")`, TinyDesk Shell's LittleFS mount point | `td_fs_stdio("<cwd>/tinydesk_fs")` |
| `extra` | `"Shell:     TinyDesk Shell <version>"` | the same with `TD_WITH_TDSH` |
| `desktop_files` | not set | not set |
| `kiosk_app` | board key `kiosk.app`, unless `board_maintenance_start()` | environment variable `TINYDESK_KIOSK` |
| `kiosk_splash`, `kiosk_hide_status` | board keys `kiosk.splash` and `kiosk.status = 0`, with `kiosk_app` | `TINYDESK_KIOSK_SPLASH`, `TINYDESK_KIOSK_HIDE_STATUS` set |
| `mouse`, `keyboard` | the ESP32-S3 with `usb.host = 1` (`usb_hid.c`); not set elsewhere | not set |

### Identification

| Member | Meaning |
|---|---|
| `platform` | Platform name shown by About. |
| `chip` | Chip description shown by About. |
| `sdk_version` | SDK version shown by About. |
| `extra` | Optional extra line for About, for example the embedded shell version. |

The strings are read when About draws; they must stay valid.

### Files for a new Desktop

`desktop_files` lists files the port puts on every new Desktop, for example
the board's credits or notes. When a user's `~/Desktop` folder is created
(the first time that user's desktop starts), TinyDesk writes `Welcome.txt`
and then these files into it. Like `Welcome.txt`, they are written once per
user: Desktops that already exist do not get them, and a user may change or
delete them. Builds that leave `desktop_files` `NULL` get only `Welcome.txt`.

```c
typedef struct {
    const char *name;   /* "Credits.txt"; NULL ends the list */
    const char *text;   /* the file's contents */
} td_desktop_file_t;
```

The array ends with an entry whose `name` is `NULL`. A name must be a plain
file name: entries with `/` or `\`, an empty name, `.`, `..` or a `NULL`
`text` are skipped. The array and the strings must stay valid; they are read
again whenever a new user's Desktop is created.

```c
static const td_desktop_file_t s_desk_files[] = {
    {"Credits.txt",
     "MyBoard port of TinyDesk\n"
     "\n"
     "Port:      Jane Doe (https://github.com/janedoe)\n"
     "Testing:   Sam Roe\n"
     "\n"
     "TinyDesk:  https://github.com/tinydesk-project/tinydesk\n"
     "           (its contributors: CONTRIBUTORS.md)\n"},
    {NULL, NULL},
};
...
s_info.desktop_files = s_desk_files;
```

A suggested format for a port's `Credits.txt`, in plain text so it opens in
the Editor and reads well on an 80-column screen:

- first line: what this is, such as `MyBoard port of TinyDesk`;
- a blank line, then one line per person or group: the role, padded to a
  column, then the name and, if they like, a link;
- a blank line, then where the port and TinyDesk come from.

Keep lines under 78 characters. The file is a starting point for the user,
not a licence notice: licence texts stay in the port's repository.

### Memory and CPU

| Member | Used by | Meaning |
|---|---|---|
| `free_heap` | About, System Monitor, Task Manager | Free heap in bytes. |
| `min_free_heap` | System Monitor | Lowest free heap since boot. |
| `total_heap` | System Monitor, Task Manager | Heap size. The memory bars need both `free_heap` and a non-zero `total_heap`. |
| `psram_free`, `psram_total` | About, System Monitor, Task Manager | External RAM (PSRAM) on the heap. Leave `psram_total` `NULL` or return 0 when there is none. When `psram_total()` returns non-zero, `psram_free` must be set too: the apps call it without checking. |
| `task_count` | System Monitor | Number of tasks or threads. |
| `cpu_mhz` | System Monitor, Task Manager | CPU clock. |
| `chip_temp` | System Monitor | The chip's own temperature (its die, not the room) in tenths of a degree Celsius; return false when it cannot be read. |
| `tasks` | Task Manager | Fill up to `max` entries of `out` and return how many. See below. |

With external RAM (`psram_total` non-zero), the three heap figures must cover internal RAM only. It is the scarce part (task stacks, DMA buffers, the SSH server's start check), and the apps show internal RAM and PSRAM separately.

### td_task_info_t

```c
typedef struct {
    char name[16];
    char state;                /* 'R' running, 'r' ready, 'B' blocked, 'S' suspended, 'D' deleted */
    uint8_t priority;
    int8_t core;               /* pinned to this core, -1 any */
    uint32_t stack_free;       /* bytes of stack never used so far */
    int16_t cpu_tenths;        /* share of all cores since the previous call, in 0.1 %; -1 unknown */
} td_task_info_t;
```

| Field | Meaning |
|---|---|
| `name` | Task name, NUL-terminated (15 characters at most). |
| `state` | `'R'` running, `'r'` ready, `'B'` blocked, `'S'` suspended, `'D'` deleted. |
| `priority` | Current priority. |
| `core` | Core the task is pinned to, or -1 for any. |
| `stack_free` | Bytes of stack never used so far (the high-water mark). |
| `cpu_tenths` | Share of all cores used since the previous `tasks()` call, in tenths of a percent (0 to 1000), or -1 when unknown. The first call has no previous sample and returns -1. |

The Task Manager calls `tasks()` about once a second with `max` = 40. It computes the total CPU load as 100 % minus the share of tasks whose name starts with `"IDLE"`, so name the idle tasks that way. The ESP32 implementation allocates its previous sample on the first call, so the cost is only paid while the Task Manager is open.

### Clock and time zone

| Member | Used by | Meaning |
|---|---|---|
| `time_now` | taskbar clock, Date & time | Store the current time in seconds since 1970-01-01 UTC and return `true`; return `false` while the clock is not set (the taskbar then shows `"--:-- no clock"`). Called on every frame: keep it cheap. The ESP32 port treats times before 1700000000 (November 2023) as not set. |
| `time_set` | Date & time | Set the system clock. Optional; without it the window says the time comes from the host. The window only lets root call it, and only while automatic time is off. |
| `time_auto_get` | Date & time | Whether the clock is set automatically from the network. Its presence also decides whether the window shows the "Set time automatically" checkbox and "Sync now" button. |
| `time_auto_set` | Date & time | Turn automatic time on or off; kept across reboots. Root only in the window. |
| `time_sync` | Date & time, clock menu | Start a network time sync now and return at once: `true` if it was started, `false` if it cannot run (no network, already running). |
| `get_tz` | clock, `td_time_of_day()`, Date & time | Store the current desktop user's offset from UTC in seconds (east positive) and return `true`. Called on every frame through the clock: cache the value (both ports re-read their file at most every few seconds, and at once when `td_session_user()` changes). Without it, UTC is shown. |
| `set_tz` | Date & time zone picker | Store a new offset for the current desktop user. Any user may change their own time zone. |

The time zone is per user (`td_session_user()` and `td_session_home()` from [apps.md](apps.md#sessions) tell whose); the clock itself and automatic time are device-wide.

### User accounts

| Member | Meaning |
|---|---|
| `user_exists` | True if a user with that name exists. Asked by Start > Switch user... before the password prompt. |
| `authenticate` | True if the password is right for that user. When set, the desktop starts behind the Sign in window, and the session code adds "Switch user..." and "Sign out" to the start menu. |
| `users` | Optional: accounts with roles and rights, for the Sign in list, the Users app, rights checks and remembered login. |

Without them the desktop always belongs to root (unless the terminal backend reports another shell user).

```c
#define TD_USER_ROLE_USER 0
#define TD_USER_ROLE_ADMIN 1
#define TD_USER_ROLE_ROOT 2
#define TD_RIGHT_USERS    (1u<<0)   /* Manage accounts (with the Admin role) */
#define TD_RIGHT_SYSTEM   (1u<<1)   /* System settings */
#define TD_RIGHT_NETWORK  (1u<<2)
#define TD_RIGHT_HARDWARE (1u<<3)   /* Hardware / I/O */
#define TD_RIGHT_SHARED   (1u<<4)   /* the /shared folder */
#define TD_RIGHT_TIME     (1u<<5)   /* Clock */
#define TD_RIGHT_APPS     (1u<<6)   /* Apps / Terminal */
#define TD_RIGHT_ALL      127u

typedef struct {char name[32];int role;uint32_t rights;} td_user_info_t;
typedef struct {
    int (*list)(int index,td_user_info_t *out);
    int (*role)(const char *username);
    uint32_t (*rights)(const char *username);
    uint32_t (*generation)(const char *username);
    int (*create)(const char *actor,const char *name,const char *password,int role,uint32_t rights);
    int (*access_set)(const char *actor,const char *name,int role,uint32_t rights);
    int (*password_set)(const char *actor,const char *name,const char *old_password,const char *password);
    uint32_t (*remembered)(char *name,int cap);
    bool (*remember_allowed)(const char *name);
    uint32_t (*login)(const char *name,const char *password);
    int (*boot_user_set)(const char *actor,const char *name);
    int (*require_password)(const char *actor);
    int (*hint_get)(const char *name,char *out,int cap);
    int (*hint_set)(const char *actor,const char *name,const char *hint);
} td_user_ops_t;
#define TD_HINT_MAX 64
```

`list` fills the account at `index` (0 on success, otherwise past the
last). `generation` is a number that changes when the account's password
is replaced or the account is made again; the desktop locks a session
whose generation changed. `login` checks a password and returns the
account's generation (0: wrong), and records the proof for a remembered
login. `remembered` returns the generation of the remembered account (and
its name), or 0. `create`, `access_set`, `password_set`, `boot_user_set`
and `require_password` act as the account `actor` and return 0 or a
negative `errno` value (`-EACCES` when that account may not). The
optional `hint_get` and `hint_set` read and set an account's password
hint (up to `TD_HINT_MAX - 1` bytes; `""` none or remove): anyone may read
it, it is shown by Sign in's **Hint**; the account itself and whoever may
manage it set it. Without them Sign in has no Hint button. The ESP ports
implement it with TinyDesk Shell's account store.

### Settings storage

| Member | Meaning |
|---|---|
| `settings_load` | Fill `len` bytes of `data` from storage and return `true`, or return `false` if nothing (or a blob of another size) is stored. `td_settings_apply_saved()` calls it with `len` = 8 to read the device defaults for users who have no `~/.tinydesk_settings` file yet. |
| `settings_save` | Store `len` bytes of `data`. The current apps never call it: per-user settings are written as files through `fs` (see `td_settings_save()` in [apps.md](apps.md#settings)). |

### Filesystem, network, updates

`fs`, `net` and `ota` point to the ops tables described below. Each may be `NULL`: Files and the Editor then show "No filesystem on this platform.", the Network app and the taskbar network indicator are absent or show "n/a", and Software Update shows `no_ota_text` (one line per `
`: why there are no updates and how to update instead; the 4 MB ESP32 port sets it), or, when that is `NULL` too, "Updates over the network are for the boards."

## Filesystem

```c
typedef struct {
    const char *root;   /* starting directory, e.g. "/fs" */

    int (*list)(const char *dir,
                void (*fn)(const char *name, bool is_dir, uint32_t size, void *user),
                void *user);
    int (*read)(const char *path, char *buf, int cap);
    int (*remove)(const char *path);
    int (*mkdir)(const char *path);
    int (*write)(const char *path, const char *data, int len);
    int (*rename)(const char *from, const char *to);
    int (*exists)(const char *path);
    int (*read_at)(const char *path, uint32_t offset, char *buf, int cap);
    int (*append)(const char *path, const char *data, int len);
} td_fs_ops_t;

#define TD_FS_READ 0     /* actions for a path guard */
#define TD_FS_WRITE 1
#define TD_FS_LIST 2
#define TD_FS_EXISTS 3
#define TD_FS_REMOVE 4
```

Minimal file access for Files, the Editor, the Desktop folder, per-user settings and the MQTT app's saved configuration. Paths are real paths: `root` followed by the path the shell sees (`"/fs" + "/root/Desktop/a.txt"`). Paths passed in are at most about 160 bytes.

| Member | Required | What it must do |
|---|---|---|
| `root` | yes | Top of the tree the apps may use, without a trailing `/`. Users' homes are `root + "/root"` and `root + "/home/<user>"`. |
| `list` | yes | Call `fn(name, is_dir, size, user)` once per entry of `dir`, not for `"."` or `".."`. Return the number of entries, or -1 when the folder cannot be read. `name` only needs to be valid during the call. |
| `read` | yes | Read up to `cap` bytes from the start of the file into `buf` (no NUL is added) and return the byte count, or -1 on error. The Editor asks for one byte more than it can hold to detect files that are too large. |
| `write` | yes | Create or replace the file with exactly `len` bytes (`len` may be 0). Return 0 on success. |
| `mkdir` | yes | Create one folder (the parent exists or the call fails). Return 0 on success. It is called on existing folders too; failing then is fine. |
| `remove` | yes | Delete a file, or a folder with everything in it. Return 0 on success. |
| `rename` | yes | Rename or move a file or folder. Return 0 on success. Should fail rather than overwrite an existing target. |
| `exists` | no | Return 1 if the path exists, else 0. Used to refuse names that are taken; without it, `rename` and `write` decide. |
| `read_at` | no | Read up to `cap` bytes starting `offset` bytes into the file; return the byte count (0 at the end) or -1. The Image Viewer reads large files with it. |
| `append` | no | Add `len` bytes to the end of the file, making it if missing; return 0 on success. **Copy to Apps** copies files larger than memory with `read_at` and `append`. |

The apps check `fs` and a few members (Files checks `list`, the Editor `read` and `write`), but Files and the Desktop menus then call `write`, `mkdir`, `remove` and `rename` without checking, so implement all members except `exists`. All calls are made on the UI task and should finish in milliseconds; a flash filesystem is fine.

### td_fs_stdio

Every port uses the same implementation on top of the C library and `<dirent.h>` (on the ESP32 the ESP-IDF VFS maps these calls to LittleFS).

```c
const td_fs_ops_t *td_fs_stdio(const char *root);
```

Declared in `ports/common/td_fs_stdio.h`. Returns a static table (one per program; a second call replaces the root). `root` is copied into a 128-byte buffer, so longer paths are cut. `td_fs_stdio_guard(allow)` installs a check that every call passes through: `allow(path, action)` with a `TD_FS_*` action, false refuses it. The ports install `td_session_path_allowed()` ([apps.md](apps.md#rights-and-sign-in)), so the apps cannot reach paths the desktop's account may not. Behaviour: `remove` deletes folders recursively up to 8 levels deep; `rename` refuses to overwrite an existing target; `write` writes the whole file in one `fopen("wb")`; `mkdir` uses mode 0755 (`_mkdir` on Windows); entries whose `stat()` fails are listed as files of size 0.

## Network

```c
typedef struct {
    bool wifi_up;              /* associated and has an IP address */
    char ssid[33];
    int rssi;                  /* dBm */
    char wifi_ip[16];
    bool eth_present;          /* Ethernet hardware enabled */
    bool eth_up;
    char eth_ip[16];
    bool busy;                 /* a scan / connect / disconnect is running */
    char message[64];          /* outcome of the last operation */
} td_net_status_t;

enum { TD_SERVER_SSH = 0, TD_SERVER_FTP = 1 };

typedef struct {
    char ssid[33];
    int rssi;
    bool secure;               /* needs a password */
    bool saved;                /* credentials are stored */
} td_wifi_ap_t;
```

`td_net_status_t` fields:

| Field | Meaning |
|---|---|
| `wifi_up` | Associated with an access point and has an IP address. |
| `ssid` | Network name while connected. |
| `rssi` | Signal strength in dBm. The taskbar shows 3 bars from -60 dBm, 2 from -72 dBm, else 1. |
| `wifi_ip` | Wi-Fi address as dotted text. |
| `eth_present` | Ethernet hardware is enabled. |
| `eth_up` | Ethernet link is up with an address. |
| `eth_ip` | Ethernet address as dotted text. |
| `busy` | A scan, connect or disconnect is running. |
| `message` | Outcome of the last operation, shown by the Network app (for example "Connected to home"). |

`td_wifi_ap_t` fields: `ssid` (network name), `rssi` (dBm), `secure` (needs a password), `saved` (credentials are stored, so connecting needs no password).

### td_net_ops_t

```c
typedef struct {
    void (*status)(td_net_status_t *out);
    bool (*scan)(void);
    int (*scan_results)(td_wifi_ap_t *out, int max);
    bool (*connect)(const char *ssid, const char *password);
    bool (*disconnect)(void);
    bool (*forget)(const char *ssid);

    bool (*server_status)(int which, int *port, int *clients);
    bool (*server_set)(int which, bool on);

    bool (*telnet_enabled)(void);
    void (*telnet_enable)(bool on);
    const char *(*telnet_peer)(void);
} td_net_ops_t;
```

Network control for the Network app and the taskbar indicator. Every call returns at once; slow work runs in the background and its outcome shows up in `status()`.

| Member | Required | What it must do |
|---|---|---|
| `status` | yes | Fill `*out` (it is zeroed by the taskbar, but fill every field). Called on every frame by the taskbar and every second by the Network app: cache the figures (the ESP32 port refreshes them once a second). |
| `scan` | no | Start a Wi-Fi scan in the background. Return `true` if it started. |
| `scan_results` | no | Copy up to `max` results of the last scan (the Network app passes 24) and return how many, or -1 while the scan is still running. |
| `connect` | no | Start connecting to `ssid`. `password` `NULL` means use the saved credentials; otherwise the network and password are saved. Return `true` if the attempt started. Wi-Fi passwords come from `td_passwordbox()`, so at most `TD_TEXT_MAX - 1` characters (47 on the ESP32 builds). |
| `disconnect` | no | Start disconnecting. |
| `forget` | no | Delete a saved network. Return `false`, with an explanation in `status().message`, when the current user may not (on the ESP32 port saved networks belong to the user who added them and root's are shared). |
| `server_status` | no | For `which` = `TD_SERVER_SSH` or `TD_SERVER_FTP`: return `true` while that server runs and store its port and client count (either pointer may be `NULL`). |
| `server_set` | no | Start or stop a server. Return `true` if the request was taken; report the result in `status().message`. The ESP32 port does this on the calling (UI) task, because the SSH server checks for 96 KiB of free internal RAM when it starts. Both `server_status` and `server_set` are needed for the Network app's server buttons. |
| `telnet_enabled` | no | Whether remote desktop over Telnet is on. |
| `telnet_enable` | no | Turn remote desktop over Telnet on or off. |
| `telnet_peer` | no | Address of the connected Telnet client, or `NULL`. |

Which display a Telnet client gets is a [Displays](#displays) setting.

Implementation: `ports/esp_idf/app/net_esp.c` (`net_esp_ops()`), on top of TinyDesk Shell's Wi-Fi and Ethernet modules (so the Network app and TinyDesk Shell share saved networks) and `telnet.c`. Scans and connects run in a worker task with a 6 KiB stack; one job at a time. The host build has no network backend unless one is installed with `td_host_use_net()` (declared in `ports/common/td_host_hal.h`).

## Displays

```c
enum
{
    TD_DISPLAY_OFF = 0,
    TD_DISPLAY_BOARD = 1,    /* the board's own link (USB serial, UART, LCD) */
    TD_DISPLAY_TELNET = 2,
    TD_DISPLAY_SERIAL = 3,   /* a second serial port */
    TD_DISPLAY_SOURCES = 4
};
typedef struct
{
    bool (*source_available)(int display, int src);
    int (*source)(int display);
    void (*set_source)(int display, int src);
    const char *(*source_label)(int src);
    void (*status)(int display, char *buf, int cap);
    void (*save_layout)(const td_screens_layout_t *layout);
    int (*rotation)(int display);
    bool (*set_rotation)(int display, int degrees);
} td_display_ops_t;
```

`td_sysinfo()->displays` (optional) is what **Settings → Displays** shows
and changes beside the core's layout ([core.md](core.md#second-screen)):
which terminal display 1 (0) and display 2 (1) are. Without it the app
still arranges a second screen the port attached itself.

| Member | What it must do |
|---|---|
| `source_available` | Whether `display` can be shown on `src` (display 1: the board or Telnet; display 2: off, Telnet, or a second serial port when one is configured). |
| `source`, `set_source` | The source in use; setting one saves it and applies at once (the port attaches or detaches display 2). Needs `TD_RIGHT_SYSTEM`. |
| `source_label` | A label, such as the board's link or `Serial 2: UART1, TX 17 / RX 18`. |
| `status` | What the display shows now: a Telnet client's address and user, "waiting for a client", "no terminal answers yet". |
| `save_layout` | Keep `td_screens_layout_t` for the next start (the app sets the layout itself first). |
| `rotation`, `set_rotation` | A display the board drives itself (the CrowPanel's LCD): its orientation, 0, 90, 180 or 270 degrees, saved for the next start. `rotation` returns -1 when the display's terminal decides (a PuTTY window). |

Implementation: `ports/esp_idf/app/hal_mux.c` (`hal_mux_displays()`):
NVS keys `disp_src1`, `disp_src2`, `disp_layout`
([config files](../shell/config-files.md)); Telnet clients from
`telnet.c` (two at once when both displays are Telnet); the second serial
port from the board keys `display2.*`
([board configuration](../guide/board-config.md#second-display-serial)).
The CrowPanel's `link_crowpanel.c` provides `board_display_rotation()` and
`board_set_display_rotation()` (NVS `lcd_rot`; `display_set_rotation()`
in `display.c` turns the frame).

## Software update

```c
enum { TD_OTA_IDLE, TD_OTA_CHECKING, TD_OTA_INSTALLING, TD_OTA_DONE, TD_OTA_FAILED };

typedef struct {
    char version[32];          /* running firmware, e.g. "0.1.0" */
    char built[32];            /* "Sep 24 2026 10:12:03" */
    char sdk[32];              /* "v5.3.1" */
    char running[17];          /* slot names, e.g. "ota_0" */
    char next[17];             /* where an update goes */
    bool on_trial;             /* new version not confirmed yet: a crash rolls back */
    bool can_roll_back;        /* the other slot holds a working version */
    char other_version[32];    /* its version */
} td_ota_info_t;

typedef struct {
    int state;                 /* TD_OTA_* */
    int percent;               /* -1 while the size is unknown */
    uint32_t done, total;      /* bytes */
    uint32_t bytes_per_s;
    char new_version[32];      /* of the image being checked / installed */
    char new_built[32];
    char message[96];
} td_ota_status_t;

typedef struct {
    bool valid;                /* a check succeeded */
    bool newer;                /* and it is newer than the installed version */
    char version[24];
    char date[12];             /* "2026-10-02" */
    char url[200];             /* its app image: give it to start() */
    uint32_t size;             /* bytes, 0 unknown */
    char notes[128];           /* the release page */
    char error[96];            /* why the last check failed, "" when it did not */
    uint32_t checks;           /* finished checks so far (to see a new result) */
} td_ota_release_t;

typedef struct {
    void (*info)(td_ota_info_t *out);
    bool (*start)(const char *source, bool check_only);
    void (*status)(td_ota_status_t *out);
    void (*cancel)(void);
    void (*restart)(void);
    bool (*roll_back)(void);   /* boot the other slot's version */

    /* Official releases (optional, NULL: none). */
    bool (*check_official)(bool quiet);
    void (*official)(td_ota_release_t *out);
    bool (*auto_check)(void);
    void (*set_auto_check)(bool on);
    void (*notified)(char *out, int cap);
    void (*set_notified)(const char *version);
} td_ota_ops_t;
```

States:

| Value | Meaning |
|---|---|
| `TD_OTA_IDLE` | Nothing running. |
| `TD_OTA_CHECKING` | Reading and checking an image without installing it. |
| `TD_OTA_INSTALLING` | Writing an image to the other slot. |
| `TD_OTA_DONE` | Finished; after an install, a restart runs the new version. |
| `TD_OTA_FAILED` | Failed or cancelled; `message` says why. |

`td_ota_info_t` describes the installed firmware: `version`, `built` (build date and time), `sdk`, the `running` and `next` slot names, `on_trial` (the running version is new and not confirmed yet, so a crash rolls back), `can_roll_back` and the other slot's `other_version`.

`td_ota_status_t` is the progress of the current job: `state`, `percent` (-1 while the size is unknown), `done` and `total` bytes, `bytes_per_s`, the image's `new_version` and `new_built`, and a `message` for the user.

| Member | What it must do |
|---|---|
| `info` | Fill `*out` (zero what is unknown). |
| `start` | Start checking (`check_only`) or installing from `source`, an `http://` or `https://` URL or a real file path, in the background. Return `false` if a job is already running or `source` is empty. |
| `status` | Copy the current progress. Polled every 250 ms by the Software Update window. |
| `cancel` | Ask the running job to stop; the installed version stays. |
| `restart` | Restart the device (to run the new version). |
| `roll_back` | Boot the other slot's version. Return `false` if that is not possible; on success it normally does not return. |

Official releases (optional; Software Update shows *Check for official updates* and *Check daily and notify me* only when the port sets them):

| Member | What it must do |
|---|---|
| `check_official` | Look up the newest official release in the background; `false` if a job is already running. With `quiet` (the daily check) `status()` stays as it is; otherwise it works like a check (`TD_OTA_CHECKING`, then a `message`). |
| `official` | Copy the result: `valid` once a check succeeded (kept when a later one fails, which only sets `error`), `newer`, `version`, `date`, `url` (the app image, for `start()`), `size`, `notes`; `checks` counts finished checks. |
| `auto_check`, `set_auto_check` | The *check daily and notify me* setting (default on). |
| `notified`, `set_notified` | The version the user was last told about, so each version is announced once. |

Board settings (optional, `NULL`: none):

| Member | What it must do |
|---|---|
| `unsaved_settings` | How many board settings (pins) only the running firmware has, built in from its `board.conf`; 0 when none. Firmware built without them, such as an official release, would start without them. |
| `save_settings` | Save them on the device, so any firmware keeps them; `false` on failure. Describes the outcome in `msg` (`cap` bytes). |

When both are set and `unsaved_settings()` is above 0, Software Update asks before *Install*: *Save and install*, *Install anyway* or *Cancel*. The ESP port uses `tdsh_board_unsaved()` and `tdsh_board_save_builtin()` (`board save`).

The ESP port reads `update-desktop-<board>.json` from the web installer's site (or the board key `update.url`, or an address a feed moved it to with `moved`), keeps the setting in NVS (`td_update`) and resolves the feed's `image` relative to the feed. Software Update checks 2 minutes after start-up and then every 24 hours (every 30 minutes while checks fail).

When `ota` is set, the first six members must be set; the app calls them without checking. The app only lets root start, cancel or roll back; everyone can look.

Implementation: `ports/esp_idf/app/ota_esp.c` (`ota_esp_ops()`), shared with the `ota` shell command. An update is written to the slot that is not running, checked (image format, chip, SHA-256) and made the boot slot. HTTPS is checked against the ESP-IDF certificate bundle. The new version runs on trial after the restart until `ota_esp_boot_ok()` confirms it (30 s after start-up, or on a requested restart); a crash before that makes the bootloader go back. The hosts have no `ota`.

## Downloads

```c
enum { TD_FETCH_IDLE, TD_FETCH_RUNNING, TD_FETCH_DONE, TD_FETCH_FAILED };
typedef struct
{
    int state;                 /* TD_FETCH_* */
    uint32_t done, total;      /* bytes; total 0 while unknown */
    char message[96];          /* why it failed */
} td_fetch_status_t;
typedef struct
{
    bool (*start)(const char *source, const char *dest, uint32_t max);
    void (*status)(td_fetch_status_t *out);
    void (*cancel)(void);
} td_fetch_ops_t;
```

`start()` copies `source`, an `http://` or `https://` URL or a real file
path (also `file://`), into the real file path `dest`, replacing it, in
the background; one download at a time (false while one runs). HTTPS is
checked against the port's trusted certificates and redirects are
followed. More than `max` bytes (0: no limit) fails. A failed or cancelled
download removes `dest`. Poll `status()`; the Store does every 100 ms.

The ESP port (`ports/esp_idf/app/fetch_esp.c`) uses `esp_http_client`
with the ESP-IDF certificate bundle (up to 4 redirects) in a short-lived
task; the PC programs (`ports/common/host_fetch.c`) copy files themselves
and hand URLs to `curl` in a thread.

`store_url` is the catalog the Store reads (`apps/store.c`): a URL or a
real file path of a `catalog.txt`; the packages come from the same folder.
NULL: `https://tinydesk-project.github.io/tinydesk-apps/catalog.txt`.

### Kiosk mode

`kiosk_app` is a Python script's shell path (`/root/hmi.py`, or a `.mpy`).
When it is set, the desktop starts in [kiosk mode](apps.md#kiosk-mode) and
runs it full screen. It must be set before `td_apps_register_all()` and stay
valid. `kiosk_splash` is a picture's shell path (BMP, PPM or PGM, up to
800 x 480 pixels) shown while the app starts; `kiosk_hide_status` leaves
the screen black instead of showing the status line. The app starts after
Sign in, for an account with `TD_RIGHT_APPS`, and not for a non-root
account when the path is under `/root`.

### Mouse and keyboard

A port whose board reads a mouse or keyboard itself (not through the
terminal), such as the ESP32-S3 with a USB keyboard and mouse, sets `mouse`
and `keyboard`. The Mouse and Keyboard apps are only registered when they
are set, so they must be in place before `td_apps_register_all()`.

```c
typedef struct {
    int speed;       /* percent; 100: 8 mouse counts a column, 16 a row */
    int accel;       /* percent; 0: none, 100: fast moves go up to 4 times as far */
    int precision;   /* percent; 0: none, 100: slow moves go 0.4 times as far */
} td_mouse_settings_t;

typedef struct {
    bool (*status)(char *buf, int cap);   /* what is attached; false: nothing */
    void (*get)(td_mouse_settings_t *s);
    bool (*set)(const td_mouse_settings_t *s);   /* apply and save */
} td_mouse_ops_t;

typedef struct {
    int layout;      /* index for layout_name() */
    int delay_ms;    /* before a held key repeats */
    int rate;        /* repeats a second */
} td_keyboard_settings_t;

typedef struct {
    bool (*status)(char *buf, int cap);
    void (*get)(td_keyboard_settings_t *s);
    bool (*set)(const td_keyboard_settings_t *s);
    const char *(*layout_name)(int i);    /* NULL past the last layout */
} td_keyboard_ops_t;
```

| Member | Meaning |
|---|---|
| `status` | Writes what is attached (for example `"USB mouse 04ca:00bd, with its wheel"`) and returns `true`; `false` when nothing is plugged in. The apps call it once a second. |
| `get` | The settings in use. |
| `set` | Applies the settings at once and saves them; returns `false` when saving failed (they still apply). The ESP32-S3 saves them in `/etc/board.conf` (`usb.*`, see [Board configuration](../guide/board-config.md#usb-keyboard-and-mouse-esp32-s3)). |
| `layout_name` | The name of layout `i` (`"US (English)"`, `"German"`...), `NULL` for `i` past the last. |

The keys and mouse themselves reach TinyDesk as the bytes a terminal
would send (mixed into the port's `link_read()`), so the rest of TinyDesk
does not know where they came from. The one difference is the Windows key,
which arrives as `TD_MOD_SUPER` (see [input.md](input.md)). A port that
draws no pointer of its own calls `td_wm_set_pointer(true)` while a mouse
is attached.
