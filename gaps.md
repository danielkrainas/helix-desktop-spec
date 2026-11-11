# Gaps

Living log of missing, unclear, or underspecified contracts.

## Open gaps

- Full `org.gnome.Shell.Screenshot` interface (InteractiveScreenshot, Screenshot, ScreenshotWindow, ScreenshotArea) — present in GNOME Shell; needs extraction.
- Mutter ScreenCast / RemoteDesktop D-Bus interfaces (private-ish but used by portals).
- DisplayConfig write side: `ApplyMonitorsConfig` method values (verify / temporary / persistent), its `logical_monitors` and `properties` arguments, and the confirmation/revert flow; `SetBacklight`, gamma and CTM semantics; keys in `Backlight` dictionaries. Helix reports `ApplyMonitorsConfigAllowed` = false and does not implement them yet.
- Which input counts as "user activity" for IdleMonitor (e.g. whether pointer motion without buttons does). Helix counts every input event.
- Session startup ordering in GNOME 51 (gnome-session's systemd targets) is not documented here. Helix uses its own simple launcher; components announce readiness by owning a bus name or calling `SessionManager.Setenv` (the compositor sets `WAYLAND_DISPLAY`).
- GSettings key tables exist for `org.gnome.desktop.interface`, `.a11y.interface` and `.wm.preferences`, but were described from gsettings-desktop-schemas 46.1 (the version available when extracted), not 51. Keys added since (e.g. `accent-color` in 47) are missing. Regenerate against 51.
- GSettings key tables for other high-traffic schemas (`org.gnome.desktop.wm.keybindings`, peripherals).
- SessionManager client and inhibitor objects: the object path shape (Helix currently uses `/org/gnome/SessionManager/Client<n>` and `/org/gnome/SessionManager/Inhibitor<cookie>`) and the interfaces exported at those paths (`org.gnome.SessionManager.Client`, `.ClientPrivate`, `.Inhibitor`) are not in the contract. Helix returns the paths but does not export objects there yet.
- SessionManager D-Bus error names are unspecified. Helix uses `org.gnome.SessionManager.Error.{NotInInitialization,AlreadyInitialized,NotInRunning,NotRegistered,InvalidCookie,InvalidArgs,Inhibited,General}` pending a contract.
- SessionManager inhibitor/client lifetime: Helix drops a caller's clients and inhibitors when its bus connection goes away (so a crashed app can't block logout forever). Not specified by the contract.
- SessionManager `CanShutdown` / `CanReboot` / `CanSuspend` return a `u` availability whose values are undocumented. Helix does not implement them yet.
- SessionManager session-restore API (`RegisterRestore`, `DeletedInstanceIds`, `UnregisterRestore`): semantics of `reason`, `instance_id` and `cleanup_ids` are undocumented. Helix sets `RestoreSupported` to false and does not implement them.
- How `Shutdown` / `Reboot` reach the system (gnome-session uses `org.freedesktop.login1`, which is not in the spec). Helix currently only ends the session.
- Where `Setenv` values must be propagated (session children, D-Bus/systemd activation environment). Helix currently only records them.
- Sources of truth for `SessionIsActive` (logind) and `SessionIsLocked` (screensaver/shell). Helix currently reports active=true, locked=false.
- Portal backend requirements beyond the standard XDG portal interfaces (GNOME-specific impl details).
- Settings portal: namespaces beyond `org.freedesktop.appearance` are undocumented, yet GTK reads `org.gnome.desktop.interface` (and others) through the portal when sandboxed. Which namespaces and keys a GNOME-compatible backend must expose is not in the spec. Helix exposes a fixed set of GSettings schemas.
- Settings portal: how GNOME maps `org.gnome.desktop.interface accent-color` names to the `(ddd)` sRGB `accent-color` value. Helix does not report `accent-color` yet.
- Settings / Session portal backends: the value of the `version` property each backend should report is undocumented. Helix reports 1.
- Inhibit portal "query end" flow: requires a SessionManager query-end phase (gnome-session's `QueryEndSession` client protocol), which is not in the spec. Helix monitors report running → ending only.
- Shell extension API surface (explicitly deferred; internal and unstable).
- `org.gnome.Shell` `ShellVersion`: Helix reports the targeted GNOME version (`51.0`) so clients that probe the shell see the surface Helix implements. Extension tooling also reads it; since `org.gnome.Shell.Extensions` is not implemented, extension installs fail cleanly, but whether to report a Helix version instead is open.
- `org.gnome.Shell` grab flags (`grabFlags` values beyond 0) are undocumented.
- `org.gnome.Shell.xml` declares `UngrabAccelerator` / `UngrabAccelerators` without an out argument; upstream GNOME Shell may return a `b` success value. Verify against the GNOME 51 introspection and fix the contract if so. Helix follows the contract as written.
- How Helix's separate compositor and shell processes share keyboard grabs for `GrabAccelerator` (in GNOME both live in one process). Helix records grabs but does not deliver `AcceleratorActivated` yet.
