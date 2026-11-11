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

## Monitors and windows (Helix)

- The primary monitor is the one DisplayConfig marks primary: the one
  configured so, else the leftmost one that is on.
- Shell surfaces (layer surfaces) that do not name a monitor go on the
  primary monitor. Their reserved zones (panels) shrink the work area of
  that monitor only.
- New windows open on the monitor under the pointer, cascading from the
  top-left corner of its work area.
- Maximizing fills the work area of the monitor a window mostly covers;
  fullscreen covers that monitor, or the monitor the client asks for.
- Popup menus are kept inside the monitor their window mostly covers.
- When a monitor goes away (unplugged or turned off), its layer surfaces
  are closed (`closed` event), so the shell creates them again on the
  remaining monitors, and windows that no longer touch any monitor move
  onto the primary one. Maximized and fullscreen windows follow the size
  of the monitor they are on.

## Window switching (Helix)

Keys of `org.gnome.desktop.wm.keybindings` (and `-backward` variants):

- `switch-applications` (defaults include Alt+Tab and Super+Tab) opens the
  application switcher: one entry per application (windows with the same
  app id), ordered by each application's most recently focused window. A
  window without an app id is an entry of its own. Entries show the
  application's name (from its desktop file, else the app id) and how many
  windows it has. Activating an entry activates the application's most
  recently focused window.
- `switch-windows` opens the window switcher: one entry per window, most
  recently focused first.
- `switch-group` (defaults Alt+Above_Tab and Super+Above_Tab) opens the
  window switcher for the focused window's application only.
- In each switcher the entry after the current one is selected (the last
  one, backward); pressing the keys again moves the selection (backward
  with the `-backward` keys or Shift). Releasing the modifier activates
  the selected entry; Escape cancels. With nothing else to switch to, the
  keys do nothing.
- `Above_Tab` in an accelerator means the key above Tab, whatever symbol
  the layout gives it (the grave accent on a US layout).
- `current-workspace-only` of `org.gnome.shell.app-switcher` and
  `org.gnome.shell.window-switcher` limits the switchers to the active
  workspace (`contracts/gsettings/workspaces.md`); `app-icon-mode` is
  ignored, as Helix shows no thumbnails or icons.
- The application launcher lists open windows (matching the search on
  title or app id) above the apps; choosing one activates it.
- Activating raises the window, gives it focus, and brings it back if it
  was minimized.
- The compositor tells the shell about windows (app id, title, focus,
  workspace) and the workspaces (how many, which is active), and
  activates windows, switches workspaces and moves windows between them on
  request, over Helix's private `helix_window_list_v1` Wayland protocol,
  offered only to the shell it launched.

## Screen capture (Helix)

The shell serves `org.gnome.Shell.Screenshot`, but the pixels are the
compositor's. The shell asks for captures over Helix's private
`helix_screenshot_v1` Wayland protocol, offered only to the shell the
compositor launched: it names the target (every monitor, an area in
layout coordinates, or the focused window), whether to draw the pointer,
and a file descriptor; the compositor writes a PNG image to it and
reports the captured rectangle, or why it failed. Captures are refused
while the session is locked. The shell can also hand the compositor a PNG
image to put on the clipboard (`image/png`); the compositor then owns the
selection and serves it to Wayland and X11 clients until something else
replaces it. Ordinary clients have no way to capture the
screen; sandboxed apps use the Screenshot portal.

## Global keys (Helix)

The shell binds its keys through Helix's private `helix_keybindings_v1`
protocol. Bound keys do nothing while the session is locked, except
those the shell marks as working while locked (volume and brightness).
