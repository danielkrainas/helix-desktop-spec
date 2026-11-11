# org.helix.ScreenCastChooser — behavioral notes

Helix-private. In GNOME, the portal backend (xdg-desktop-portal-gnome)
draws the screen-sharing chooser itself; in Helix the shell does, so the
dialog looks like the rest of the shell and appears above everything.

- Bus name `org.gnome.Shell`, object `/org/helix/ScreenCastChooser`.
- `ChooseSource(app_id, monitors, options)`: `monitors` lists
  `(connector, label)` pairs to offer. `options`:
  - `windows` (`b`): also offer the open windows (the shell knows them;
    they are listed by title and application, most recently used first);
  - `monitors` (`b`, default true): offer the monitors;
  - `remember` (`b`): the app asked to share again later without asking;
    the dialog says so.

  The shell asks "Share your screen?" naming the app (its desktop file
  name, else the app id, else "An application"). Choosing a monitor
  returns `(true, 1, connector, 0)`; choosing a window `(true, 2, "",
  window)`, where `window` is the id the compositor uses for it in the
  window list (`helix_window_list_v1`, and the `window-id` of
  `org.gnome.Mutter.ScreenCast` `RecordWindow`). Cancel, Escape or the
  session locking returns `(false, 0, "", 0)`.
- One question at a time: a call while one is open fails with
  `org.freedesktop.DBus.Error.Failed`.
- Only the owner of `org.freedesktop.impl.portal.desktop.helix` may call
  it (`org.freedesktop.DBus.Error.AccessDenied` otherwise).
