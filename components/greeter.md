# Greeter (login screen)

Helix's login screen, so that a Helix system needs neither GDM nor GNOME
Shell. It runs under **greetd**, a minimal login daemon: greetd owns the
seat's login, PAM and session start; Helix supplies the greeter that
greetd runs. (GDM cannot be kept without GNOME Shell: GDM's greeter is
GNOME Shell in a login mode, and distributions make the GDM package
depend on it.)

## How it runs

- greetd starts the greeter on its VT as its greeter user (`_greetd` on
  Debian and Ubuntu), in a logind session of class `greeter`, with
  `GREETD_SOCK` naming greetd's socket.
- Helix's greetd configuration (`/etc/greetd/helix-greeter.toml`, used
  through a drop-in for `greetd.service` installed by the `helix-greeter`
  package, so greetd's own `config.toml` is left alone) runs
  `/usr/libexec/helix/helix-greeter-session`. That starts a private
  session bus and, on it, the compositor on the bare hardware
  (`helix-comp --backend drm --exit-with-shell --shell helix-greeter`).
  No session manager runs.
- `--exit-with-shell`: when the shell program exits with status 0 the
  compositor exits too (instead of restarting it). The greeter exits
  that way once greetd has accepted the session to start; greetd starts
  the session when the greeter's process tree is gone.

## The greetd protocol

Messages both ways are a 32-bit length in native byte order followed by
that many bytes of UTF-8 JSON. Requests (`type`):

- `create_session` (`username`): begin a login for that user.
- `post_auth_message_response` (`response`, optional): answer the last
  authentication message.
- `start_session` (`cmd`: argument list, `env`: `NAME=value` list): the
  session to start once the greeter has exited.
- `cancel_session`: give up the login being set up.

Replies: `success`; `error` (`error_type` `auth_error` for wrong
credentials, else `error`, and a `description`); `auth_message`
(`auth_message_type` `visible`, `secret`, `info` or `error`, and the
message text). Authentication may ask any number of questions in any
order; the greeter makes no assumptions about them.

## What the screen shows

On every monitor a dark background (a vertical gradient), and on the
primary monitor (see "The monitor" below):

- **Clock** at the top: the time and the date (`%A, %B %-d`) in the
  system locale (see "Language" and "Clock" below).
- **User list**: the people who can log in, by display name, most
  recently logged in first. A user is listed when AccountsService
  (`org.freedesktop.Accounts`, `ListCachedUsers`) knows them and they are
  not system accounts; without AccountsService, the accounts in
  `/etc/passwd` with a UID from 1000 to 59999 and a login shell (not
  `nologin` or `false`). With `user-list = false` in the configuration,
  or when no users are found, a **Username** field is shown instead.
  The last user to log in through the greeter is selected at start.
  Each user shows their picture, AccountsService's `IconFile` (PNG or
  JPEG, up to 4 MB) cropped to a circle, as GDM does; without one, a
  coloured circle with their initials, as on the lock screen. A list
  longer than the screen scrolls to keep the selection in view.
- Choosing a user (click, Return, or arrows then Return) begins a login
  (`create_session`). Each question greetd passes on is shown as a field
  labelled with the message text: `secret` questions hide what is typed
  (a dot per character), `visible` ones show it. `info` and `error`
  messages are shown under the field and answered at once (no response).
  Return answers. Escape (or **Back**) cancels the login and returns to
  the user list.
- `auth_error` shows "Sorry, that didn't work. Please try again." and
  starts the login again for the same user; other errors show their
  description.
- **Session** button (bottom right): the sessions installed in
  `/usr/share/wayland-sessions` (desktop entries; `Hidden` or
  `NoDisplay` true are left out; `TryExec` must exist, looked up in
  `PATH` when relative), by their name in the system language
  (`Name[xx]`, see "Language"). The session follows the selected user,
  as GDM's does: the session that user last started through the greeter,
  else the one AccountsService remembers for them (`Session`, else
  `XSession`, as GDM records it), when installed; else the last session
  started through the greeter, else the Helix session, else the first.
  A session picked from the menu stays picked while the selection moves.
  X11 sessions (`/usr/share/xsessions`) are not offered: greetd
  does not start an X server, so they would need a wrapper that starts
  one (helixdesktop/desktop#76).
- **Power** (bottom left): "Restart…" and "Power Off…" through logind
  (`org.freedesktop.login1.Manager` `Reboot` / `PowerOff`, interactive),
  offered when `CanReboot` / `CanPowerOff` answer `yes` or `challenge`.
  Each asks first ("Restart this computer?" / "Power off this computer?"
  with Cancel and the action, focus on the action), as the lock screen
  does.
- Optional **banner** text under the clock (`banner` in the
  configuration).

Keyboard focus starts in the user list (or the username field); Tab
moves between the list or field, the session button, the accessibility
button and the power buttons. The look follows the lock screen's
(centered column, text fields as in "Text fields" in
`components/README.md`).

## Accessibility

An **Accessibility** button (a person symbol, accessible name
"Accessibility") sits left of the Session button. It opens a menu, as
the Session button does (Up/Down move, Return or Space toggle, Escape,
Tab or a click elsewhere close it), with a switch for each of:

- **High Contrast**: the screen as the shell draws high contrast
  (`contracts/gsettings/a11y-interface.md`): a black background instead of the gradient,
  white text, buttons, rows, fields and menus outlined in white, the
  selected user outlined, the focus ring white and set apart from the
  outlines.
- **Large Text**: text 1.25 times its size (GNOME's Large Text factor),
  and the heights and gaps that hold text (rows, fields, buttons, menu
  items) grown alike so that nothing is cut off.
- **Screen Reader**: the greeter starts Orca (`orca`, found in `PATH`)
  as its own child process with its own environment, so Orca uses the
  greeter's session bus and the accessibility bus found through it;
  turning it off ends that process (by its process id: never
  `orca --replace`, which would end other screen readers). Alt+Super+S
  turns it on and off from anywhere on the screen, as in GNOME. When the
  greeter exits it ends Orca, so that greetd can start the session.
- **Screen Keyboard**: a keyboard drawn by the greeter at the bottom of
  the login screen, between the column and the bottom bar (the clock
  and the column move up to make room; the user's picture is left out
  of the password question). Letters (QWERTY) with Shift (one capital),
  a page of digits and punctuation (its Shift: the rest of ASCII's
  punctuation and a few common signs), comma, period, Space, Backspace
  and Return. Its keys type into the greeter's own list and fields
  only, so no virtual-keyboard protocol is used.

*GNOME does not document this; Helix's choice:* the menu stays open
while its switches are toggled; the four settings are remembered in
the greeter's state file (`accessibility`: `high_contrast`,
`large_text`, `screen_reader`, `screen_keyboard`; absent: off) as soon
as they change, and apply from the next start; a power action's
question goes beside its buttons while the keyboard is shown. A
**magnifier** is not offered on the login screen.

**The screen on the accessibility bus.** The greeter publishes itself
over AT-SPI (`contracts/dbus/org.a11y.atspi.md`), as the shell does,
whether or not the screen reader is on: one window, "Login screen",
holding the time and date (labels), the banner, the users (a list of
list items, the selected one selected), the username field (an entry)
or the question's field (password text for secret questions, its text
one "•" per character typed; an entry for visible ones; not sensitive
while greetd checks), the messages (labels announced when they
change), Back, the power buttons and the confirmation question with
its buttons, the Session button (expandable; expanded while its menu is
open) and its menu (radio menu items, the current session checked), the
Accessibility button and its menu (check menu items, checked when on),
and the screen keyboard's keys (push buttons; Shift checked while on).
The focus follows the greeter's keyboard focus (a menu's highlighted
item while it is open). Each object's action does what a click at its
place does. `HELIX_NO_ATSPI` set: the greeter stays off the
accessibility bus.

## Language

The login screen runs before anyone logs in, so it follows the system
locale (`components/i18n.md`): the environment greetd gives it when that
names a locale (`LANG`, `LC_ALL` or `LC_MESSAGES`), else
`/etc/locale.conf` (systemd's, as `localectl set-locale` writes it), else
`/etc/default/locale` (Debian's). Its own words ("Username", "Back",
"Restart…", the power questions, the retry message) come from the
`helix-desktop` catalogue; month and weekday names and the time format
from the locale. A right-to-left language mirrors the screen
(`components/i18n.md`, "Right-to-left languages").

Each session's name is the entry's localized `Name[xx]` for the system
locale, chosen as the Desktop Entry Specification orders them
(`lang_COUNTRY@MODIFIER`, `lang_COUNTRY`, `lang@MODIFIER`, `lang`), else
`Name`.

## Clock

The time is 12- or 24-hour from, in order:

1. `clock-format` in the configuration (`12h` or `24h`);
2. `org.gnome.desktop.interface clock-format` as the greeter user reads
   it (the schema's default, vendor overrides and system dconf
   databases: the login screen has no user settings of its own);
3. the locale's own time format (12-hour when its `LC_TIME` time format
   uses AM/PM, as `en_US` does).

*GNOME does not document this; Helix's choice:* the order above. The
patterns are translatable (`%H:%M`, `%l:%M %p`, the date's
`%A, %B %-d`), so a language orders them its own way.

## Starting the session

Once authentication succeeds, the greeter sends `start_session` with:

- `cmd`: one string, the entry's `Exec` line. greetd does not treat `cmd`
  as an argument list: it joins the items with spaces and runs
  `/bin/sh -c '<source /etc/profile and ~/.profile>; exec <joined>'`, so a
  single string is interpreted by that shell as the desktop entry
  specification allows (arguments, quoting, `env NAME=value` prefixes).
  (Splitting it into `/bin/sh`, `-c`, `exec …` becomes `exec /bin/sh -c
  exec …` once joined, a shell that runs a bare `exec` and exits without
  starting anything.) When `/usr/bin/systemd-cat` exists the line is
  prefixed with `/usr/bin/systemd-cat --identifier=<session id> --`:
  greetd keeps no session output, so this is how the session's messages
  (and those of everything it starts) reach the journal, as they do under
  GDM. *GNOME does not document this; Helix's choice.* The greeter's own
  compositor logs the same way, as `helix-greeter`;
- `env`: `XDG_SESSION_TYPE=wayland`, `XDG_SESSION_DESKTOP` (the entry's
  file name without `.desktop`), `DESKTOP_SESSION` (the same), and
  `XDG_CURRENT_DESKTOP` (the entry's `DesktopNames`, `;` replaced by `:`)
  when it has one.

It remembers the user, the session, and that session as the user's own
in its state file (`$XDG_STATE_HOME/helix-greeter/state.json` of the
greeter user, i.e. under `/var/lib/greetd`), then exits with status 0.
*GNOME does not document this; Helix's choice:* the greeter does not
write the session to AccountsService (`SetSession` needs an
administrator's authorization, which the greeter user lacks; GDM's
daemon does it as root), so it keeps its own per-user memory and reads
AccountsService's as a fallback.

## The monitor

The login screen (clock, user list, buttons) is on the primary monitor:
the logical monitor the compositor's `org.gnome.Mutter.DisplayConfig`
`GetCurrentState` marks primary (on the greeter's private session bus),
found among the Wayland outputs by connector name (`wl_output.name`).
The greeter asks again when a monitor comes or goes and after every
`MonitorsChanged` (the compositor may bring back a remembered layout
after the greeter started). Without an answer, the first monitor
announced. Other monitors show the background.

Which monitor the compositor makes primary follows its remembered
configurations (`contracts/dbus/org.gnome.Mutter.DisplayConfig.md`,
"Remembered configurations"): the greeter user's own file, then the
system-wide `/etc/xdg/helix/monitors.toml`, for the monitors connected
now; with neither, its default (the first monitor). Users' own layouts
can't be used: their home directories are usually closed to other users,
and nobody is logged in yet.

### Setting the login screen's layout

`/usr/libexec/helix/helix-login-monitors`, run as root through `pkexec`
(polkit action `org.helixdesktop.login-screen-monitors`; administrator
authentication, kept for a while in an active session), puts a layout
into `/etc/xdg/helix/monitors.toml`. It takes no arguments and reads one
configuration on standard input, in the remembered configurations' format
with exactly one `[[configuration]]`: the layout for the monitors
connected now. Settings (Displays, "Use This Layout for the Login
Screen") sends the layout shown now, as `GetCurrentState` describes it.
The helper:

- reads at most 64 KiB, and refuses anything but exactly one
  configuration with only the keys of the format;
- refuses a set of monitors that is not written as the compositor writes
  it (sorted, no repeats, connector names of letters, digits, `-`, `_`,
  `.`, no control characters) and a layout that would not be valid for
  those monitors (the rules under `ApplyMonitorsConfig`, each monitor
  taken to have the mode chosen for it);
- replaces the configuration for the same set of monitors in the file and
  keeps the others; a file that can't be read is left alone and nothing is
  written;
- writes the file anew from what it parsed (never a copy of its input)
  into a new file next to it, mode 0644, owned by root, then renames it
  over the old one.

Exit status: 0 written, 1 invalid input, 2 the file could not be read or
written, 64 arguments given; `pkexec` itself exits 126 when the
authentication was dismissed and 127 when it was refused.

The same can be done by hand (as root): copy a user's
`~/.config/helix/monitors.toml`, or only its `[[configuration]]` for the
monitors connected at the login screen, to
`/etc/xdg/helix/monitors.toml` (mode 0644).

*GNOME does not document this; Helix's choice:* GDM documents only
copying a user's `monitors.xml` into GDM's configuration directory by
hand. Helix reads a system-wide file the administrator can set from
Settings through a polkit-checked helper. The helper takes the layout
from the caller instead of reading the user's home directory as root, so
nothing under the user's control is opened with root's rights; since it
needs an administrator anyway, it trusts the layout no more than the
rules above.

## Configuration

`/etc/helix/greeter.toml` (optional):

```toml
user-list = true          # false: always ask for the username
banner = "Authorized use only."   # text under the clock
accent = "blue"           # the accent color (focus rings, the confirming button)
clock-format = "24h"      # 12h or 24h; unset: see "Clock"
```

`accent` takes the names of `org.gnome.desktop.interface accent-color`
(`blue`, `teal`, `green`, `yellow`, `orange`, `red`, `pink`, `purple`,
`slate`); the login screen has no user whose settings it could follow.
It is drawn with the shades of `components/README.md`, "Accent color".
Unset or an unknown name: blue (the unknown name is logged).

## Where GDM differs

*GNOME does not document this; Helix's choice:* GDM's behaviour here is
not documented, and the greeter does these its own way:

- the clock: `clock-format` from the greeter's configuration, the
  system's `org.gnome.desktop.interface` defaults or the locale (see
  "Clock"), as there is no user whose setting it could read;
- the session: read from AccountsService but not written there; the
  greeter's own per-user memory comes first (see "Starting the
  session").

## Not yet done

- Orca on the login screen is tested over AT-SPI on a private bus only,
  not yet speaking on real hardware through greetd (the greeter's
  private session bus must be able to start the accessibility bus).

## Installing and switching

The `helix-greeter` package depends on `greetd` and `helix-desktop`. It
does not change which display manager is active; switching is the
administrator's step (for example `systemctl disable gdm3` then
`systemctl enable greetd`), and switching back works the same way.

Packaged for Debian and Ubuntu only: greetd's greeter user (`_greetd`
there) and the greetd binary's path differ on Fedora and Arch Linux, and
their packages are not written yet (helixdesktop/desktop#78).
