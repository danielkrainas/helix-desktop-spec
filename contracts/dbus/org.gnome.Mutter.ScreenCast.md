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

- `Version` is 2. `RecordMonitor`, `RecordWindow`, `RecordArea` and
  `RecordVirtual` are implemented; the per-stream `Start` / `Stop` fail
  with `org.freedesktop.DBus.Error.NotSupported`. `RecordWindow` takes
  `window-id` (`t`): the window's id in Helix's window list (see
  `org.helix.ScreenCastChooser.md`); an unknown id fails with
  `org.freedesktop.DBus.Error.InvalidArgs`. Window streams have a `size`
  parameter (the window's size in layout coordinates) and no position.
  `RecordArea` records the rectangle (layout coordinates) clipped to the
  monitor it mostly covers; its `position` and `size` parameters are the
  clipped rectangle, and an area on no monitor (or empty) fails with
  `org.freedesktop.DBus.Error.InvalidArgs`. `RecordVirtual` adds a virtual
  monitor of 1920×1080 at scale 1, to the right of the other monitors,
  for as long as the stream lives; windows can be moved onto it (by
  dragging, `move-to-monitor-*`) like any monitor, but it is not listed
  in DisplayConfig. Its parameters have `position`, `size` and
  `output-name` (`Virtual-N`). Cursor mode 2 adds PipeWire cursor metadata
  (`SPA_META_Cursor`) to every buffer: the pointer's position and hotspot
  in stream pixels and, as a `spa_meta_bitmap`, a BGRA image (straight
  alpha, at most 64×64, cropped) of the pointer as Helix draws it (see
  "Pointer image" in `contracts/wayland/README.md`); `id` is
  0 while the pointer is hidden or off the source. When only the pointer
  moved, the last frame is sent again with new metadata. `is-recording` and
  `disable-animations` are accepted and ignored; a `remote-desktop-session-id`
  ties the session to that remote desktop session
  (`org.gnome.Mutter.RemoteDesktop.md`), an unknown one fails with
  `org.freedesktop.DBus.Error.InvalidArgs`.
- Only the owner of `org.freedesktop.impl.portal.desktop.helix` (Helix's
  portal backend, which asks the user first) may call these methods;
  others get `org.freedesktop.DBus.Error.AccessDenied`. A session ends
  when its creator leaves the bus.
- Streams are BGRx frames, at most 30 per second and only when the
  picture changed (a new consumer gets a whole frame at once), in
  shared-memory buffers: a monitor upright at its pixel size, or a window cropped to its
  window geometry at the highest scale among the monitors it is on (popup
  menus inside it included). When that size changes (a mode change, the window
  resized) the stream's format changes with it. While the session is
  locked frames are black; while a shared window is hidden (on another
  workspace, minimized) no frames are sent. The session ends when its
  monitor goes away or its window closes.
- The compositor also serves `org.helix.ScreenCastState` at
  `/org/gnome/Mutter/ScreenCast`: `Active` (`b`) is true while any session
  is started, and emits `PropertiesChanged`. The shell shows a "Sharing"
  indicator in the panel while it is true.
