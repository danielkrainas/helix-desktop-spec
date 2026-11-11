# Wayland contracts (GNOME 51)

Notes on protocols and usage expected for a GNOME-compatible session under Helix.

## Core expectations

- Standard Wayland protocols sufficient for normal desktop operation (xdg-shell, xdg-decoration, primary selection, etc.).
- `wlr-layer-shell-unstable-v1` (or equivalent) for shell surfaces (panels, notifications, OSD).
- Mutter/GNOME-specific private protocols are out of scope for initial compatibility; public portals and D-Bus are preferred.

## GNOME 51 notes

- Support for `ext-background-effect-v1` (blur) was added in Mutter; apps must explicitly request it.
- Screen casting and remote desktop use PipeWire + portals (`org.freedesktop.portal.ScreenCast`, etc.) rather than direct compositor APIs where possible.
- Idle inhibition and related behavior route through Mutter's IdleMonitor and session inhibitors.

## Portals (preferred path)

Many session interactions (screenshots, screencasts, inhibit, settings) should prefer XDG Desktop Portals:

- `org.freedesktop.portal.Screenshot`
- `org.freedesktop.portal.ScreenCast`
- `org.freedesktop.portal.Inhibit`
- `org.freedesktop.portal.Settings`

Helix should implement the portal backends needed for GTK/libadwaita apps to function without modification.

Detailed protocol XML or behavioral notes will be added as gaps are identified.

## Session locking

The lock screen uses the standard `ext-session-lock-v1` protocol: while
locked, the compositor shows only the lock client's surfaces and sends input
only to them, and stays locked if the lock client dies.

## Window switching (Helix)

- `switch-applications` and `switch-windows` (and their `-backward`
  variants) in `org.gnome.desktop.wm.keybindings` open the window
  switcher; defaults include Alt+Tab and Super+Tab. The switcher lists the
  open windows, most recently focused first, with the previously focused
  window selected; pressing the keys again moves the selection
  (backward with the `-backward` keys or Shift). Releasing the modifier
  activates the selected window; Escape cancels. With a single window
  there is nothing to switch to.
- The application launcher lists open windows (matching the search on
  title or app id) above the apps; choosing one activates it.
- Activating raises the window, gives it focus, and brings it back if it
  was minimized.
- The compositor tells the shell about windows (app id, title, focus) and
  activates them on request over Helix's private `helix_window_list_v1`
  Wayland protocol, offered only to the shell it launched.
