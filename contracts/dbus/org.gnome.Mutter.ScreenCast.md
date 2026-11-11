# org.gnome.Mutter.ScreenCast — behavioral notes

Companion to `org.gnome.Mutter.ScreenCast.xml`. The compositor's screen
cast API: it records monitors into PipeWire video streams. Portal backends
(and, in GNOME, gnome-remote-desktop) use it.

## Bus name and objects

- Bus name: `org.gnome.Mutter.ScreenCast`
- `/org/gnome/Mutter/ScreenCast`: `org.gnome.Mutter.ScreenCast`
- Sessions and streams get object paths of their own, returned by
  `CreateSession` and the `Record*` methods.

## Flow

1. `CreateSession(properties)` returns a session.
2. `RecordMonitor(connector, properties)` (and the other `Record*`
   methods) on the session return streams; their `Parameters` describe
   the source: `position` (`(ii)`) and `size` (`(ii)`) in layout
   coordinates, and `output-name` (`s`) for monitors.
3. `Start` on the session starts recording. Each stream then emits
   `PipeWireStreamAdded(node_id)` with the id of its PipeWire node, which
   clients connect to.
4. `Stop` ends the session: its streams end, `Closed` is emitted and the
   objects go away. The compositor may also end a session itself (e.g.
   when the recorded monitor goes away), with `Closed`.

Cursor modes (`cursor-mode`, `u`): 0 hidden, 1 embedded in the frames,
2 sent as PipeWire metadata.

## Helix

- `Version` is 2. `RecordMonitor` is implemented; `RecordWindow`,
  `RecordArea`, `RecordVirtual` and the per-stream `Start` / `Stop` fail
  with `org.freedesktop.DBus.Error.NotSupported`. Cursor mode 2 is not
  supported (`org.freedesktop.DBus.Error.InvalidArgs`); `is-recording` and
  `disable-animations` are accepted and ignored; a `remote-desktop-session-id`
  is refused (no RemoteDesktop).
- Only the owner of `org.freedesktop.impl.portal.desktop.helix` (Helix's
  portal backend, which asks the user first) may call these methods;
  others get `org.freedesktop.DBus.Error.AccessDenied`. A session ends
  when its creator leaves the bus.
- Streams are BGRx frames of the monitor's current mode (its pixel size,
  upright), at most 30 per second, in shared-memory buffers. While the
  session is locked they are black.
- The compositor also serves `org.helix.ScreenCastState` at
  `/org/gnome/Mutter/ScreenCast`: `Active` (`b`) is true while any session
  is started, and emits `PropertiesChanged`. The shell shows a "Sharing"
  indicator in the panel while it is true.
