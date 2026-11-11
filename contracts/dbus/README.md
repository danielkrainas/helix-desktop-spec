# D-Bus contracts

Initial public contracts extracted for GNOME 51 compatibility.

## Included

- `org.gnome.SessionManager.xml` — session lifecycle, inhibit, logout/shutdown
- `org.gnome.SessionManager.ClientPrivate.xml`, `org.gnome.SessionManager.Inhibitor.xml` — session client and inhibitor objects
- `org.gnome.SessionManager.EndSessionDialog.xml` — logout/shutdown confirmation (served by the shell)
- `org.gnome.Mutter.DisplayConfig.xml` — multi-monitor configuration
- `org.gnome.Mutter.IdleMonitor.xml` — idle time and activity watches
- `org.gnome.Shell.xml` — shell control (overview, OSD, accelerators)
- `org.gnome.ScreenSaver.xml` — lock state and locking
- `org.freedesktop.Notifications.xml`, `org.gtk.Notifications.xml` — notifications
- `org.freedesktop.Application.xml` — application activation (called by the shell)
- `org.freedesktop.impl.portal.{Settings,Inhibit,Request,Session}.xml` — XDG portal backend interfaces (xdg-desktop-portal 1.22.1)
- `org.freedesktop.login1.xml` — the subset of systemd-logind Helix calls (power, sessions, sleep inhibitors); not implemented by Helix

Behavioral notes that XML cannot express live in companion `.md` files
(`org.gnome.SessionManager.md`, `org.gnome.Mutter.IdleMonitor.md`,
`org.gnome.Mutter.DisplayConfig.md`, `org.gnome.Shell.md`, `org.gnome.ScreenSaver.md`, `org.freedesktop.Notifications.md`,
`org.gtk.Notifications.md`, `org.freedesktop.Application.md`,
`org.gnome.SessionManager.EndSessionDialog.md`,
`org.freedesktop.impl.portal.md`, `org.freedesktop.login1.md`).

## Notes

These are clean-room extractions of the public introspection interfaces from upstream GNOME 51 and xdg-desktop-portal sources.

The `org.gnome.*` interface shapes (members, argument types and directions,
property access) were checked against the upstream 51.0 tags on 2026-11-11. They represent the contract surface, not implementation code.

Additional interfaces (ScreenCast, RemoteDesktop, portals, Shell.Screenshot, etc.) will be added as the living spec grows.
