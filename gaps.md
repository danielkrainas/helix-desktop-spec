# Gaps

Living log of missing, unclear, or underspecified contracts.

## Open gaps

- Full `org.gnome.Shell.Screenshot` interface (InteractiveScreenshot, Screenshot, ScreenshotWindow, ScreenshotArea) — present in GNOME Shell; needs extraction.
- Mutter ScreenCast / RemoteDesktop D-Bus interfaces (private-ish but used by portals).
- Complete GSettings schema XML for high-traffic keys (`org.gnome.desktop.interface`, `org.gnome.desktop.wm.keybindings`, peripherals).
- SessionManager client and inhibitor objects: the object path shape (Helix currently uses `/org/gnome/SessionManager/Client<n>` and `/org/gnome/SessionManager/Inhibitor<cookie>`) and the interfaces exported at those paths (`org.gnome.SessionManager.Client`, `.ClientPrivate`, `.Inhibitor`) are not in the contract. Helix returns the paths but does not export objects there yet.
- SessionManager D-Bus error names are unspecified. Helix uses `org.gnome.SessionManager.Error.{NotInInitialization,AlreadyInitialized,NotInRunning,NotRegistered,InvalidCookie,InvalidArgs,Inhibited,General}` pending a contract.
- SessionManager inhibitor/client lifetime: Helix drops a caller's clients and inhibitors when its bus connection goes away (so a crashed app can't block logout forever). Not specified by the contract.
- SessionManager `CanShutdown` / `CanReboot` / `CanSuspend` return a `u` availability whose values are undocumented. Helix does not implement them yet.
- SessionManager session-restore API (`RegisterRestore`, `DeletedInstanceIds`, `UnregisterRestore`): semantics of `reason`, `instance_id` and `cleanup_ids` are undocumented. Helix sets `RestoreSupported` to false and does not implement them.
- How `Shutdown` / `Reboot` reach the system (gnome-session uses `org.freedesktop.login1`, which is not in the spec). Helix currently only ends the session.
- Where `Setenv` values must be propagated (session children, D-Bus/systemd activation environment). Helix currently only records them.
- Sources of truth for `SessionIsActive` (logind) and `SessionIsLocked` (screensaver/shell). Helix currently reports active=true, locked=false.
- Portal backend requirements beyond the standard XDG portal interfaces (GNOME-specific impl details).
- Shell extension API surface (explicitly deferred; internal and unstable).
