# Gaps

Living log of missing, unclear, or underspecified contracts.

## Open gaps

- Full `org.gnome.Shell.Screenshot` interface (InteractiveScreenshot, Screenshot, ScreenshotWindow, ScreenshotArea) — present in GNOME Shell; needs extraction.
- Mutter ScreenCast / RemoteDesktop D-Bus interfaces (private-ish but used by portals).
- DisplayConfig: D-Bus error names for rejected `ApplyMonitorsConfig` calls (Helix uses `org.freedesktop.DBus.Error.InvalidArgs` for stale serials and invalid configurations); per-monitor `properties` keys in `ApplyMonitorsConfig` (Helix ignores them); whether a confirm-or-revert step follows a change and who shows it (Helix has none); fractional scales; `SetBacklight` / `ChangeBacklight`, gamma and CTM semantics; keys in `Backlight` dictionaries. Helix does not implement the backlight, gamma and CTM methods yet.
- Which input counts as "user activity" for IdleMonitor (e.g. whether pointer motion without buttons does). Helix counts every input event.
- Session startup ordering in GNOME 51 (gnome-session's systemd targets) is not documented here. Helix uses its own simple launcher; components announce readiness by owning a bus name or calling `SessionManager.Setenv` (the compositor sets `WAYLAND_DISPLAY`).
- GSettings key tables for other high-traffic schemas (`org.gnome.desktop.wm.keybindings`, peripherals).
- SessionManager D-Bus error names are unspecified. Helix uses `org.gnome.SessionManager.Error.{NotInInitialization,AlreadyInitialized,NotInRunning,NotRegistered,InvalidCookie,InvalidArgs,Inhibited,General}` pending a contract.
- SessionManager inhibitor/client lifetime: Helix drops a caller's clients and inhibitors when its bus connection goes away (so a crashed app can't block logout forever). Not specified by the contract.
- SessionManager `CanShutdown` / `CanReboot` / `CanSuspend` return a `u` availability whose values are undocumented. Helix does not implement them; its shell asks logind (`CanPowerOff` etc., whose answers are documented) instead.
- SessionManager session-restore API (`RegisterRestore`, `DeletedInstanceIds`, `UnregisterRestore`): semantics of `reason`, `instance_id` and `cleanup_ids` are undocumented. Helix sets `RestoreSupported` to false and does not implement them.
- Whether `Setenv` values must also reach the systemd user manager's environment (GNOME runs its session as systemd user units). Helix does not use systemd units and only updates the D-Bus activation environment.
- `org.gtk.Notifications` `ActionInvoked` signal (added in GNOME 51): when GNOME Shell emits it relative to activating the app over `org.freedesktop.Application` is undocumented. Helix activates the app and does not emit it.
- `org.gnome.SessionManager.EndSessionDialog` `Open` type values beyond 0 (logout), 1 (power off) and 2 (restart) are undocumented (GNOME also has upgrade-related dialogs). Helix only uses 0–2.
- Portal backend requirements beyond the standard XDG portal interfaces (GNOME-specific impl details).
- Settings portal: namespaces beyond `org.freedesktop.appearance` are undocumented, yet GTK reads `org.gnome.desktop.interface` (and others) through the portal when sandboxed. Which namespaces and keys a GNOME-compatible backend must expose is not in the spec. Helix exposes a fixed set of GSettings schemas.
- Settings portal: how GNOME maps `org.gnome.desktop.interface accent-color` names to the `(ddd)` sRGB `accent-color` value. Helix does not report `accent-color` yet.
- Settings / Session portal backends: the value of the `version` property each backend should report is undocumented. Helix reports 1.
- Shell extension API surface (explicitly deferred; internal and unstable).
- `org.gnome.Shell` grab flags (`grabFlags` values beyond 0) are undocumented.
