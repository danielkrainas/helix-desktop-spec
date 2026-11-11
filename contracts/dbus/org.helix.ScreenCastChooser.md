# org.helix.ScreenCastChooser — behavioral notes

Helix-private. In GNOME, the portal backend (xdg-desktop-portal-gnome)
draws the screen-sharing chooser itself; in Helix the shell does, so the
dialog looks like the rest of the shell and appears above everything.

- Bus name `org.gnome.Shell`, object `/org/helix/ScreenCastChooser`.
- `ChooseMonitor(app_id, monitors)`: `monitors` lists `(connector,
  label)` pairs. The shell asks "Share your screen?" naming the app (its
  desktop file name, else the app id, else "An application") and lists the
  monitors; choosing one returns `(true, connector)`. Cancel, Escape or
  the session locking returns `(false, "")`.
- One question at a time: a call while one is open fails with
  `org.freedesktop.DBus.Error.Failed`.
- Only the owner of `org.freedesktop.impl.portal.desktop.helix` may call
  it (`org.freedesktop.DBus.Error.AccessDenied` otherwise).
