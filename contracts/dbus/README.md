# D-Bus contracts

Initial public contracts extracted for GNOME 51 compatibility.

## Included

- `org.gnome.SessionManager.xml` — session lifecycle, inhibit, logout/shutdown
- `org.gnome.SessionManager.ClientPrivate.xml`, `org.gnome.SessionManager.Inhibitor.xml` — session client and inhibitor objects
- `org.gnome.SessionManager.EndSessionDialog.xml` — logout/shutdown confirmation (served by the shell)
- `org.gnome.Mutter.DisplayConfig.xml` — multi-monitor configuration
- `org.gnome.Mutter.IdleMonitor.xml` — idle time and activity watches
- `org.freedesktop.PolicyKit1.AuthenticationAgent.xml` — polkit authentication agent (served by the shell)
- `org.freedesktop.GeoClue2.Agent.xml` — GeoClue's location agent (served by the shell) and the Manager calls it makes
- `org.freedesktop.NetworkManager.SecretAgent.xml` — NetworkManager's secret agent (served by the shell, Wi-Fi passwords)
- `org.bluez.Agent1.xml` — BlueZ's pairing agent (served by the shell)
- `org.gnome.keyring.internal.Prompter.xml` — gcr's system prompter (served by the shell)
- `org.gnome.Shell.SearchProvider2.xml` — app search providers (called by the shell)
- `org.gnome.Shell.Screencast.xml` — screen recording (served by the shell)
- `org.gnome.Shell.xml` — shell control (overview, OSD, accelerators)
- `org.gnome.Shell.Screenshot.xml` — screenshots, area selection, color picking (served by the shell)
- `org.gnome.ScreenSaver.xml` — lock state and locking
- `org.freedesktop.Notifications.xml`, `org.gtk.Notifications.xml` — notifications
- `org.freedesktop.Application.xml` — application activation (called by the shell)
- `org.freedesktop.impl.portal.{Access,Account,AppChooser,Background,Wallpaper,Lockdown,Notification,Print,FileChooser}.xml` — more portal backend interfaces (served by `helix-portals`)
- `org.helix.PortalDialog.xml` — Helix-private: dialogs and running apps the portal backends ask the shell for
- `org.freedesktop.impl.portal.{Settings,Inhibit,Screenshot,ScreenCast,RemoteDesktop,Request,Session}.xml` — XDG portal backend interfaces (xdg-desktop-portal 1.22.1)
- `org.helix.DisplayChangeDialog.xml` — Helix-private: the shell asks whether to keep a display change (called by the compositor)
- `org.gnome.Mutter.ScreenCast.xml` — recording monitors into PipeWire streams (served by the compositor)
- `org.gnome.Mutter.RemoteDesktop.xml` — remote input, Notify* and EIS (served by the compositor)
- `org.helix.ScreenEffects.xml` — Helix-private: grayscale for screen time limits (served by the compositor)
- `org.helix.ScreenCastChooser.xml`, `org.helix.ScreenCastState.xml` — Helix-private: the shell asks what to share; the compositor says whether sharing is on
- `org.gnome.SettingsDaemon.Color.xml` — Night Light state (served by the shell)
- `org.freedesktop.IBus.xml` — the part of IBus Helix's shell calls as the input method; not implemented by Helix
- `org.mpris.MediaPlayer2.Player.xml` — the part of MPRIS Helix calls on media players (playback keys); not implemented by Helix
- `org.freedesktop.login1.xml` — the subset of systemd-logind Helix calls (power, sessions, sleep inhibitors, brightness); not implemented by Helix

Behavioral notes that XML cannot express live in companion `.md` files
(`org.gnome.SessionManager.md`, `org.gnome.Mutter.IdleMonitor.md`,
`org.gnome.Mutter.DisplayConfig.md`, `org.gnome.Shell.md`, `org.gnome.Shell.Screenshot.md`, `org.gnome.ScreenSaver.md`, `org.freedesktop.Notifications.md`,
`org.gtk.Notifications.md`, `org.freedesktop.Application.md`,
`org.gnome.SessionManager.EndSessionDialog.md`,
`org.freedesktop.impl.portal.md`, `org.freedesktop.login1.md`,
`org.helix.DisplayChangeDialog.md`, `org.mpris.MediaPlayer2.Player.md`,
`org.gnome.Mutter.ScreenCast.md`, `org.helix.ScreenCastChooser.md`,
`org.gnome.SettingsDaemon.Color.md`, `org.freedesktop.GeoClue2.Agent.md`,
`org.gnome.evolution.dataserver.md`, `org.freedesktop.NetworkManager.SecretAgent.md`, `org.bluez.Agent1.md`).

## Notes

These are clean-room extractions of the public introspection interfaces from upstream GNOME 51 and xdg-desktop-portal sources.

The `org.gnome.*` interface shapes (members, argument types and directions,
property access) were checked against the upstream 51.0 tags on 2026-11-11. They represent the contract surface, not implementation code.

Additional interfaces (RemoteDesktop, more portals, etc.) will be added as the living spec grows.
