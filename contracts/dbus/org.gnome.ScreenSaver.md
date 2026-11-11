# org.gnome.ScreenSaver — behavioral notes

Companion to `org.gnome.ScreenSaver.xml`. Served by the shell.

## Bus name and object

- Bus name: `org.gnome.ScreenSaver`
- Object: `/org/gnome/ScreenSaver`

## Semantics

"Active" means the screen shield is up; in Helix that is always the lock
screen.

- `Lock`: lock the session now. Returns without waiting for the lock to be
  in place; `ActiveChanged(true)` follows when it is.
- `GetActive`: whether the session is locked.
- `SetActive(true)`: same as `Lock`. `SetActive(false)` never unlocks a
  locked session (only authentication does); it has no effect.
- `GetActiveTime`: seconds since the session was locked, 0 when unlocked.
- `ActiveChanged(new_value)`: emitted on every lock and unlock.
- `WakeUpScreen`: asks for the screen to be woken (e.g. after it blanked).

`org.gnome.SessionManager`'s `SessionIsLocked` property follows
`ActiveChanged`.

## Idle locking

The session locks automatically when the user has been idle for
`org.gnome.desktop.session idle-delay` seconds (0 = never) plus
`org.gnome.desktop.screensaver lock-delay` seconds, if
`org.gnome.desktop.screensaver lock-enabled` is true. An active
`SessionManager` inhibitor with the idle flag (8) prevents it.

## Locking before suspend

The shell holds a logind `sleep` inhibitor in `delay` mode (see
`org.freedesktop.login1.md`). On `PrepareForSleep(true)` it locks if
`org.gnome.desktop.screensaver lock-enabled` is true, waits until the lock
screen is shown, then releases the inhibitor so the system can sleep. On
`PrepareForSleep(false)` it takes a new one.

## Unlocking

Unlocking requires authenticating as the session's user through PAM.
