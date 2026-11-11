# org.gnome.Mutter.IdleMonitor — behavioral notes

Companion to `org.gnome.Mutter.IdleMonitor.xml`.

## Bus name and objects

- Bus name: `org.gnome.Mutter.IdleMonitor`
- An `org.freedesktop.DBus.ObjectManager` at `/org/gnome/Mutter/IdleMonitor`
- The monitor for all input devices at `/org/gnome/Mutter/IdleMonitor/Core`

Clients (e.g. gnome-settings-daemon) find `Core` through the object manager.

## Semantics

- Times are milliseconds.
- `GetIdletime`: time since the last user activity.
- `AddIdleWatch(interval)`: returns a watch id (non-zero). `WatchFired(id)`
  is emitted when the idle time reaches `interval` — immediately if it
  already has. The watch then stays registered and fires again the next time
  the user is idle for `interval` after some activity. `interval` must be
  greater than zero.
- `AddUserActiveWatch`: fires `WatchFired(id)` once, at the next user
  activity, and is then removed.
- `RemoveWatch(id)`: removes a watch of either kind.
- `ResetIdletime`: counts as user activity (idle time restarts from zero and
  user-active watches fire).
- Watches belong to the caller: they are removed when the caller leaves the
  bus.
