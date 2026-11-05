# First login and users

## A freshly installed board

1. Open the board's serial port in a terminal
   ([settings](terminals.md#putty-settings)). Press a key if the screen stays
   empty.
2. The desktop starts behind a full-screen **Sign in** window. Choose
   **root** and type its password.

The factory root password is **`TinyDesk`** (capital T and D). Change it
locally before enabling remote access. Telnet starts disabled; Telnet, SSH
and FTP refuse remote authentication until the root password changes.
Enable Telnet explicitly in Network only on a trusted network: it sends
passwords in plain text and accepts root only (see [Security](../security.md)):

```text
passwd                 # in the Terminal, as root: old password, then the new one twice
```

Existing user passwords are preserved on upgrade. A saved Telnet setting is
honored only when the root password has changed.

## Signing in

The Sign in window lists the accounts with their role. Choose one, type its
password and press Enter or **Sign in**. Nothing else on the desktop takes
input until the shell has switched to that account.

* **Keyboard** (for a touch screen such as the CrowPanel's) shows large
  keys to touch, as big as the screen allows: digits, letters, **Shift**,
  **#+=** for the symbols, **Space**, **Del** and **Clear**; the Enter key
  signs in and **Hide** goes back to the list. A key types when the finger
  lifts on it; sliding off a key types nothing.
* **Hint** shows the selected account's password hint, if it has one.
  The hint is set when the account is made or its password changed
  (`useradd`, `passwd`, the Users app) and anyone at the device can read
  it, so it must not give the password away: a hint that contains the
  password is refused.
* After a wrong password the next try waits 1 second; after 5 wrong
  passwords in a row, 10 seconds each time, until one is right.
* **Start → Sign out** closes the account's windows, clears the Terminal
  and the clipboard and shows Sign in again. **Start → Switch user...**
  asks for another account's name and password.
* When an account's password changes or the account is deleted and made
  again, its open sessions end and the desktop returns to Sign in.

The PC programs have no Sign in window: their desktop belongs to root.

## Remembered login

Root can let the board start without the Sign in window. Three ways turn
it on:

* tick **Remember root on this device** when signing in as root (the box
  is only there for root; signing in as root with it unticked turns
  remembered login off);
* in the **Users** app, select an account and press **Remember login**;
  **Require password at boot** turns it off;
* `bootuser NAME` in the Terminal, as root.

The account must then sign in once with its password; a name alone is not
enough. From the next start the desktop goes straight to the welcome
screen, or to the [kiosk app](using.md#kiosk-mode). A password change, or
the account deleted and made again, ends it. `bootuser --require-password`
(or `bootuser off`) turns it off and forgets the proof; a later
`bootuser NAME` needs another password sign-in. Root may put either line
in `/root/.tdshrc.tdsh`, which runs at every start.

## Accounts

TinyDesk Shell keeps up to 12 accounts in NVS: `root` and the accounts made
with `useradd` or the Users app. Passwords are stored salted and hashed
(SHA-256, 2048 rounds). User names are 1-31 characters: letters, digits,
`_`, `-` and `.`.

Every account other than root has a role, **Admin** or **User**, and a set
of rights:

| Right (Users app) | `userrights` name | Allows |
| --- | --- | --- |
| Manage accounts | `users` | with the Admin role: create, change and delete accounts |
| System settings | `system` | changes in the Mouse and Keyboard apps, `reboot` |
| Network | `network` | Wi-Fi and network commands and the Network app's changes, `ssh` and `ftp` |
| Hardware / I/O | `hardware` | `gpio`, `hwtest` |
| Shared files | `shared` | the folder `/shared` |
| Clock | `time` | setting the system clock and automatic time |
| Apps / Terminal | `apps` | the Terminal window, `py`, Python windows and kiosk apps |

A new User gets Apps / Terminal; a new Admin gets every right except
Hardware / I/O. Accounts made by older firmware become Users with Apps /
Terminal. An Admin with Manage accounts can manage accounts whose rights it
also holds, and cannot give rights it does not have, change root or change
itself. Admins and Users stay in their own home: neither sees `/root` or
another user's home. Firmware updates, `board set`, `peek` and `poke`,
`bootuser`, the Log Viewer and Python's `machine` and native code stay
root's.

| Command | Who | What it does |
| --- | --- | --- |
| `users` | anyone | list the accounts |
| `useradd <username> [admin\|user]` | root, or an Admin with Manage accounts | create an account (asks for the password twice), a User unless `admin` is given, with the home `/home/<username>` |
| `userdel <username> [-f]` | root, or an Admin with Manage accounts | delete an account and its home; without `-f` it asks for that account's password |
| `userrole <username> [admin\|user]` | show: anyone; set: as `useradd` | show or set the role; setting it resets the rights to the role's defaults |
| `userrights <username> [+right\|-right ...]` | show: anyone; change: as `useradd` | show the rights, or add and remove them by name |
| `passwd` | anyone | change your own password (asks for the old one), then the hint: Enter keeps it, `-` removes it |
| `login <username>` | anyone | continue the session as another user (asks for their password) |
| `logout` | anyone | end the session; the console asks who logs in next |
| `bootuser [username\|--require-password]` | show: anyone; set: root | the account the console starts as and the [remembered login](#remembered-login); `--require-password` (or `off`) turns remembering off |
| `rootrecover` | the board's console only | set a new root password without the old one |

The **Users** app (Start menu) does the same in a window: **New account**,
a name, the new password twice, **Administrator** and the rights, then
**Create / Apply**; **Change password** for an existing account (your own
needs its current password). Users without Manage accounts see only their
own account and can change their own password.

`rootrecover` is restricted to the physical console. After a Telnet takeover,
reboot locally before using it; remote input can remain queued after disconnect.
A Telnet takeover is refused while recovery is prompting for a password.
See [Security](../security.md).

## Which user the desktop belongs to

The desktop belongs to the account that signed in (or the remembered one).
It changes when:

* root logs in over **Telnet**: the shared desktop belongs to root until
  that connection ends; other accounts use separate SSH shell sessions;
* you choose **Start → Switch user...** (name and password) or **Start →
  Sign out**;
* you run `login <user>` or `logout` in the Terminal.

A change closes the previous user's windows (unsaved Editor changes are
lost) and clears the Terminal, so nothing of theirs stays on screen.

| | root | Admin and User accounts |
| --- | --- | --- |
| home | `/root` | `/home/<user>` |
| Files app, FTP | the whole file system | their home; with Shared files also `/shared` |
| SFTP | the whole file system | their home |
| Wi-Fi networks | adds shared ones, may remove all | with Network: adds their own, uses their own and shared ones |
| SSH / FTP servers, Telnet switch | yes | with Network, from the board's console |
| `reboot` | yes | with System settings |
| system clock | yes | with Clock |
| Log Viewer, firmware updates, `board set`, `bootuser` | yes | no |
| theme, sizes, time zone, desktop icons | their own | their own |

`/shared` is made when the board starts. Every account with Shared files
can read and write it; homes stay private.

A user's `~/.tdshrc.tdsh` runs when the console starts as that user (for
example `wificonnect` to join Wi-Fi at boot). It is not run again on a
switch; see [Configuration files](../shell/config-files.md).

## Typical set-up

```text
useradd alice                       # as root, in the Terminal: a User with Apps / Terminal
userrights alice +network +shared   # let her join Wi-Fi and use /shared
echo wificonnect >> /home/alice/.tdshrc.tdsh
bootuser alice                      # after alice signs in once, the board starts as alice
```
