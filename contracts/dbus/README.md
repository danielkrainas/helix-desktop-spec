# D-Bus contracts

Initial public contracts extracted for GNOME 51 compatibility.

## Included

- `org.gnome.SessionManager.xml` — session lifecycle, inhibit, logout/shutdown
- `org.gnome.Mutter.DisplayConfig.xml` — multi-monitor configuration
- `org.gnome.Mutter.IdleMonitor.xml` — idle time and activity watches
- `org.gnome.Shell.xml` — shell control (overview, OSD, accelerators)
- `org.freedesktop.impl.portal.{Settings,Inhibit,Request,Session}.xml` — XDG portal backend interfaces (xdg-desktop-portal 1.22.1)

Behavioral notes that XML cannot express live in companion `.md` files
(`org.gnome.SessionManager.md`, `org.freedesktop.impl.portal.md`).

## Notes

These are clean-room extractions of the public introspection interfaces from upstream GNOME 51 and xdg-desktop-portal sources. They represent the contract surface, not implementation code.

Additional interfaces (ScreenCast, RemoteDesktop, portals, Shell.Screenshot, etc.) will be added as the living spec grows.
