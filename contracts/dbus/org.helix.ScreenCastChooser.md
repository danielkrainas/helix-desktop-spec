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
  - `control` (`as`): device names (`keyboard`, `pointer`,
    `touchscreen`) the app will also control (a remote desktop session);
    the dialog says so.

  The shell asks "Share your screen?" naming the app (its desktop file
  name, else the app id, else "An application"). Choosing a monitor
  returns `(true, 1, connector, 0)`; choosing a window `(true, 2, "",
  window)`, where `window` is the id the compositor uses for it in the
  window list (`helix_window_list_v1`, and the `window-id` of
  `org.gnome.Mutter.ScreenCast` `RecordWindow`). Cancel, Escape or the
  session locking returns `(false, 0, "", 0)`.
- `AllowRemoteControl(app_id, devices)`: a remote desktop session
  without screen sharing. `devices` lists device names as above. The
  shell asks "Allow remote control?": the app wants to control the listed
  devices, "Deny" / "Allow". Returns whether the user allowed it (Escape
  or the session locking: false).
- `DescribeWindow(window)` returns `(app_id, title)` of an open window
  (by window list id); an unknown window fails with
  `org.freedesktop.DBus.Error.InvalidArgs`.
- `FindWindow(app_id, title)` returns `(true, window)` for the open window
  with that app id and title (the most recently used when several do),
  else `(false, 0)`. Neither asks the user.
- One question at a time: a call while one is open fails with
  `org.freedesktop.DBus.Error.Failed`.
- Only the owner of `org.freedesktop.impl.portal.desktop.helix` may call
  it (`org.freedesktop.DBus.Error.AccessDenied` otherwise).
