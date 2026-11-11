# Gaps

Living log of missing, unclear, or underspecified contracts.

## Open gaps

- Full `org.gnome.Shell.Screenshot` interface (InteractiveScreenshot, Screenshot, ScreenshotWindow, ScreenshotArea) — present in GNOME Shell; needs extraction.
- Mutter ScreenCast / RemoteDesktop D-Bus interfaces (private-ish but used by portals).
- Complete GSettings schema XML for high-traffic keys (`org.gnome.desktop.interface`, `org.gnome.desktop.wm.keybindings`, peripherals).
- Exact inhibit flag bitmasks and logout mode values for SessionManager.
- Portal backend requirements beyond the standard XDG portal interfaces (GNOME-specific impl details).
- Shell extension API surface (explicitly deferred; internal and unstable).
