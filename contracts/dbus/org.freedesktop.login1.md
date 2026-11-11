# org.freedesktop.login1 — what Helix relies on

Companion to `org.freedesktop.login1.xml`: the part of systemd-logind's
documented API (system bus, `org.freedesktop.login1`) that the session
manager and the shell call. Helix does not implement this interface.

## Power

- `PowerOff(interactive)`, `Reboot(interactive)`, `Suspend(interactive)`
  ask the system to do it. With `interactive` false, logind refuses rather
  than asking the user to authenticate.
- `CanPowerOff`, `CanReboot`, `CanSuspend` answer one of:
  - `yes`: allowed without authentication
  - `challenge`: allowed after authenticating
  - `no`: not allowed
  - `na`: not supported on this system

## Sessions

- `GetSession(id)` and `GetSessionByPID(pid)` return the session object.
  The id of the caller's session is in `$XDG_SESSION_ID` when set.
- The session object's `Active` (`b`) is true while the session is in the
  foreground of its seat (e.g. not switched away from on another VT). It
  emits `PropertiesChanged`.

## Inhibitors

`Inhibit(what, who, why, mode)` returns a file descriptor; the inhibitor
lasts until every copy of it is closed.

- `what`: colon-separated list; Helix uses `sleep`.
- `mode`: `block` stops the action; `delay` holds it off until the
  inhibitor is released (bounded by logind's configured maximum delay).

`PrepareForSleep(true)` is emitted just before the system sleeps, after
delay inhibitors were asked to finish; `PrepareForSleep(false)` after it
resumes.

## Brightness

`SetBrightness(subsystem, name, brightness)` on the session object sets a
`backlight` (or `leds`) device under `/sys/class/<subsystem>/<name>` to
`brightness` (0 to the device's `max_brightness`), without the caller
needing write access to sysfs. Only a session of the seat the device
belongs to may use it. Helix's compositor uses it for the `Backlight` of
`org.gnome.Mutter.DisplayConfig` and reads the current value from
`/sys/class/backlight/<name>/brightness`.
