# org.freedesktop.Application — behavioral notes

Companion to `org.freedesktop.Application.xml`. The desktop-entry spec's
D-Bus activation interface, implemented by every GLib `GApplication`.
The shell calls it on applications (it does not serve it).

- Bus name: the application id (e.g. `org.gnome.Calculator`).
- Object path: `/` + the id with `.` replaced by `/` and `-` by `_`
  (`/org/gnome/Calculator`).
- `platform_data` keys: `activation-token` (`s`, an XDG activation token),
  `desktop-startup-id` (`s`, X11 startup notification).
- `ActivateAction(name, parameter, platform_data)`: `name` without the
  `app.` prefix; `parameter` is empty or holds one value.
