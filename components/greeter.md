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
first monitor the compositor announces (the next one if it goes away):

- **Clock** at the top: the time (`%H:%M`) and the date (`%A, %B %-d`)
  in the system locale.
- **User list**: the people who can log in, by display name, most
  recently logged in first. A user is listed when AccountsService
  (`org.freedesktop.Accounts`, `ListCachedUsers`) knows them and they are
  not system accounts; without AccountsService, the accounts in
  `/etc/passwd` with a UID from 1000 to 59999 and a login shell (not
  `nologin` or `false`). With `user-list = false` in the configuration,
  or when no users are found, a **Username** field is shown instead.
  The last user to log in through the greeter is selected at start.
  Each user shows a coloured circle with their initial (account pictures
  are not read). A list longer than the screen scrolls to keep the
  selection in view.
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
  `PATH` when relative), by `Name` (not localized). The
  last session used is selected at start, else the Helix session, else
  the first. X11 sessions (`/usr/share/xsessions`) are not offered: greetd
  does not start an X server (recorded in `gaps.md`).
- **Power** (bottom left): "Restart…" and "Power Off…" through logind
  (`org.freedesktop.login1.Manager` `Reboot` / `PowerOff`, interactive),
  offered when `CanReboot` / `CanPowerOff` answer `yes` or `challenge`.
  Each asks first ("Restart this computer?" / "Power off this computer?"
  with Cancel and the action, focus on the action), as the lock screen
  does.
- Optional **banner** text under the clock (`banner` in the
  configuration).

Keyboard focus starts in the user list (or the username field); Tab
moves between the list or field, the session button and the power
buttons. The look follows the lock screen's (centered column, text fields as
in "Text fields" in `components/README.md`).

## Starting the session

Once authentication succeeds, the greeter sends `start_session` with:

- `cmd`: `/bin/sh -c 'exec <Exec>'`, so the entry's `Exec` line is
  interpreted as the desktop entry specification allows (arguments,
  quoting, `env NAME=value` prefixes);
- `env`: `XDG_SESSION_TYPE=wayland`, `XDG_SESSION_DESKTOP` (the entry's
  file name without `.desktop`), `DESKTOP_SESSION` (the same), and
  `XDG_CURRENT_DESKTOP` (the entry's `DesktopNames`, `;` replaced by `:`)
  when it has one.

It remembers the user and session in its state file
(`$XDG_STATE_HOME/helix-greeter/state.json` of the greeter user, i.e.
under `/var/lib/greetd`), then exits with status 0.

## Configuration

`/etc/helix/greeter.toml` (optional):

```toml
user-list = true          # false: always ask for the username
banner = "Authorized use only."   # text under the clock
```

## Installing and switching

The `helix-greeter` package depends on `greetd` and `helix-desktop`. It
does not change which display manager is active; switching is the
administrator's step (for example `systemctl disable gdm3` then
`systemctl enable greetd`), and switching back works the same way.
