# Wayland contracts (GNOME 51)

Notes on protocols and usage expected for a GNOME-compatible session under Helix.

## Core expectations

- Standard Wayland protocols sufficient for normal desktop operation (xdg-shell, xdg-decoration, primary selection, etc.).
- `wlr-layer-shell-unstable-v1` (or equivalent) for shell surfaces (panels, notifications, OSD).
- Mutter/GNOME-specific private protocols are out of scope for initial compatibility; public portals and D-Bus are preferred.

## More Wayland protocols (Helix)

**Idle inhibition** (`zwp_idle_inhibit_manager_v1`). While a surface with
an inhibitor is shown (its window mapped on the active workspace and not
minimized, or a shown shell surface), the session counts as in use: the
`IdleMonitor` idle time stays at 0, so the screen neither blanks nor locks.

**Pointer constraints** (`zwp_pointer_constraints_v1`) and **relative
pointer** (`zwp_relative_pointer_manager_v1`). A lock or confinement takes
effect while its surface has pointer focus (oneshot ones end when it
loses focus; persistent ones come back with focus). While locked the
pointer stays put and only relative motion is reported; when the lock
ends the pointer moves to the surface's cursor position hint, if it set
one. While confined the pointer stays inside the confinement region.
Relative motion carries the device's unaccelerated deltas (nested: the
host's motion).

**Shortcut inhibition** (`zwp_keyboard_shortcuts_inhibit_manager_v1`).
When a surface asks, the shell asks "Allow APP to use all keyboard
shortcuts?" with **Deny** / **Allow**, naming the app (as the share dialog
does). The answer is stored in the permission store
(`org.freedesktop.impl.portal.PermissionStore`, table `gnome`, id
`shortcuts-inhibitor`, the app id with `["GRANTED"]` or `["DENIED"]`) and
not asked again; apps without an app id are asked every time and nothing
is stored. While an allowed surface has keyboard focus, Helix's own
shortcuts and the shell's bindings go to the app, except
`org.gnome.mutter.wayland.keybindings restore-shortcuts` (default
`<Super>Escape`), which ends the inhibition until the surface gets
keyboard focus again, and switching virtual terminals.

**Xwayland keyboard grabs** (`zwp_xwayland_keyboard_grab_manager_v1`,
Xwayland only). With `org.gnome.mutter.wayland xwayland-allow-grabs`
true, an X11 window whose `WM_CLASS` name or class matches an entry of
`xwayland-grab-access-rules` (`*` and `?` wildcards; entries starting with
`!` deny and win over allowing ones) may grab the keyboard: it gets all
keys, as an allowed shortcut inhibitor does, until it ends the grab or
`restore-shortcuts` is pressed.

**Foreign toplevels** (`zxdg_exporter_v2`/`zxdg_importer_v2`, and v1). An
app (typically a portal) may make its toplevel a dialog of another app's
exported toplevel: it is then placed and kept as dialogs are (see
"Window-management settings").

**Touchpad gestures** (`zwp_pointer_gestures_v1`, bare hardware). Swipes
with 3 or 4 fingers are Helix's and are not passed on: when the fingers
lift having moved at least 100 logical pixels sideways, and more sideways
than up or down, the workspace switches the way the content was pulled
(fingers moving left show the next workspace, to the right; moving right
the previous one), as GNOME does regardless of `natural-scroll`; there is
none past the first or last workspace, and a cancelled swipe, or one
while the session is locked, does nothing. Other gestures (two-finger
swipes, pinches, holds) go to the surface under the pointer.

**Frame timing.** `wp_presentation` feedback reports when a frame was
shown (the monitor's vblank, its refresh interval and sequence) or that
it was discarded. `wp_fifo_manager_v1`: a commit waiting on a FIFO barrier
is applied after the next frame of the surface's monitor.
`wp_commit_timing_manager_v1`: a commit with a target time is applied at
the first frame at or after it. Commits of windows not shown (minimized,
on another workspace) are not held back. `wp_single_pixel_buffer_manager_v1`
is offered.

**Explicit sync** (`wp_linux_drm_syncobj_manager_v1`). Offered when the
GPU Helix renders with can wait on DRM syncobj timeline points with an
eventfd (bare hardware: the GPU it drives; nested: the host GPU's render
node). A commit with an acquire point is applied once that point is
signalled; the release point is signalled once Helix no longer uses the
buffer. GPU buffers committed without explicit sync wait for their
implicit (dmabuf) fences before the commit is applied, with or without
this protocol.

## Pointer image (Helix)

The pointer shows what the app under it asks for: its own image
(`wl_pointer.set_cursor` with a surface, drawn with the hotspot it gave at
the monitor's scale; it gets frame callbacks while shown), a named cursor
(`wp_cursor_shape_manager_v1`), or nothing. Elsewhere (desktop, panels
that set none, Helix's title bars and window edges) it is the theme's
default or resize cursor.

Named cursors come from the XCursor theme
`org.gnome.desktop.interface cursor-theme`, at `cursor-size` logical
pixels (the theme's nearest size for the monitor's scale, rounded up to a
whole number); a name the theme lacks falls back to its older X name, then
to `default`; with no usable theme Helix draws a plain arrow. Animated
cursors animate. Changes to either key apply at once.

Nested, the host window shows named cursors itself (from the host's
theme); an app's own image is drawn by Helix, with the host's pointer
hidden over the window meanwhile.

Screenshots that include the pointer, and screen casts with the cursor
embedded or as metadata, show the same image Helix draws. Nested,
screenshots and embedded cursors show only an app's own image (the host
draws named cursors), while cursor metadata carries the theme's image for
named cursors.

## Color management (Helix)

Helix offers `wp_color_manager_v1` (version 1) for what it renders
faithfully: it composites in the monitors' own encoding, treating all
content as sRGB, and does not convert colors.

- Advertised: the perceptual render intent, the `parametric` feature,
  the transfer functions `srgb` and `gamma22`, and the `srgb` primaries.
  Other features (ICC, scRGB, BT.2100, `set_primaries`, `set_tf_power`,
  `set_luminances`, mastering display values) are not, and using them is
  the protocol's `unsupported_feature` error; so are other named values
  (`invalid_tf`, `invalid_primaries_named`).
- A parametric image description with those values (and, optionally,
  `max_cll` / `max_fall`) is ready at once. Its identity is the same for
  the same parameters and new for each new set; identities are never
  reused.
- A surface may set any ready description with the perceptual intent; it
  does not change how the surface is drawn (its content is already right
  for every description Helix accepts). A second
  `wp_color_management_surface_v1` for a surface is the `surface_exists`
  error; once its surface is gone the object is inert.
- Every monitor's description, and every surface's preferred one, is sRGB:
  `srgb` primaries (red 0.64/0.33, green 0.30/0.60, blue 0.15/0.06, white
  0.3127/0.3290), the `srgb` transfer function, luminances 0.2 / 80 / 80
  cd/m² (minimum / maximum / reference white) and the same target volume.
  They do not change, so `image_description_changed` and
  `preferred_changed` are never sent. A monitor that has gone away gives a
  description that fails with `no_output`.

HDR output (BT.2100 color mode), color conversion between descriptions,
and the `output-luminance` level for HDR are not implemented (see
`gaps.md`); `output-luminance` for SDR dims through the gamma ramps
(`org.gnome.Mutter.DisplayConfig.md`, "Gamma").

## GPU buffers (Helix)

Helix offers `zwp_linux_dmabuf_v1` so apps can hand it GPU buffers instead
of copying pixels through shared memory. Its default feedback (version 4
and later) names the GPU Helix renders with (its DRM render node) as the
main device, with the formats and modifiers that GPU can import, so
Mesa's and other EGL implementations find the right GPU without
`wl_drm`. When nested, that is the GPU of the host window's EGL display,
if it can be found; otherwise only the version 3 format list is offered.

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
- New windows open on the monitor under the pointer (see
  "Window-management settings").
- Maximizing fills the work area of the monitor a window mostly covers;
  fullscreen covers that monitor, or the monitor the client asks for.
- Popup menus are kept inside the monitor their window mostly covers.
- When a monitor goes away (unplugged or turned off), its layer surfaces
  are closed (`closed` event), so the shell creates them again on the
  remaining monitors, and windows that no longer touch any monitor move
  onto the primary one. Maximized and fullscreen windows follow the size
  of the monitor they are on.

## Window frames (Helix)

Like GNOME, Helix draws a frame (title bar) only for X11 windows; Wayland
apps draw their own decorations. `xdg-decoration` always answers
client-side (mutter offers no server-side decorations at all).

An X11 window gets a title bar along its top edge, outside the window's
own area, unless it asks for none (the Motif `_MOTIF_WM_HINTS` property
without decorations, as apps drawing their own title bar set), is
fullscreen, or is override-redirect (menus, tooltips). Placing,
maximizing and keeping windows inside the work area count the title bar
as part of the window, so it never ends up under the panel.

The title bar is 34 logical pixels high and shows:

- the window title (`_NET_WM_NAME`, else `WM_NAME`), centered and shortened
  with an ellipsis when it does not fit, dimmed while the window does not
  have focus;
- buttons as `org.gnome.desktop.wm.preferences button-layout` says: names
  before the colon go in the left corner, after it in the right corner, in
  order. `minimize`, `maximize` and `close` are shown, `spacer` adds a
  gap, other names (`appmenu`, `menu`, `icon`, unknown ones) are left out,
  and a repeated name is shown once. Without a colon every button goes on
  the left.

Text uses the font of `titlebar-font`, or with `titlebar-uses-system-font`
the family and size of `org.gnome.desktop.interface font-name`, drawn
bold. Colors are light, or dark with `org.gnome.desktop.interface
color-scheme` `prefer-dark`. Settings changes apply at once.

Clicks on the title bar:

| Click | Effect |
|-------|--------|
| left on a button, released on it | minimize; maximize or restore; close (`WM_DELETE_WINDOW`, as the client's own close) |
| left elsewhere | focus and raise; dragging moves the window (a maximized window is restored once the pointer has moved, under the pointer) |
| two left clicks within `org.gnome.desktop.peripherals.mouse double-click` milliseconds | `action-double-click-titlebar` |
| middle | `action-middle-click-titlebar` |
| right | `action-right-click-titlebar` |

Actions: `toggle-maximize` maximizes or restores;
`toggle-maximize-horizontally` / `-vertically` fill the work area in that
direction only (or restore); `minimize`; `lower` puts the window below
all others; `menu` opens the window menu (see "Window menu and window
keys") at the pointer; `none` and `toggle-shade` (Helix does not shade
windows) do nothing.

Around a framed window (left, right, bottom, and above the title bar) is
an invisible border `org.gnome.mutter draggable-border-width` logical
pixels wide; dragging it with the left button resizes the window from
that edge or corner, and the pointer shows the matching resize cursor.
Corners are where two borders meet, extended along each edge by the
border width.

`ScreenshotWindow` with `include_frame` true includes the title bar.

## Window menu and window keys (Helix)

The window menu opens for a window when the app asks for it
(`xdg_toplevel.show_window_menu`, e.g. a right click on a GTK header bar),
on the `activate-window-menu` key (`org.gnome.desktop.wm.keybindings`,
default `<Alt>space`; for the focused window, at its top-left corner), and
as the `menu` title bar action of X11 frames. It lists, top to bottom:

- **Minimize**;
- **Maximize**, or **Restore** for a maximized window;
- **Move** and **Resize** (see below; not for maximized or fullscreen
  windows);
- **Always on Top** (checked when on): the window stays above windows
  that are not;
- **Always on Visible Workspace** (checked when on; `toggle-on-all-workspaces`);
- **Move to Workspace Left** / **Right**, when there is one that way;
- **Move to Monitor Left** / **Right** / **Up** / **Down**, when there is a
  monitor that way;
- **Close**.

The menu closes on a choice, Escape, or a click outside it.

**Move** and **Resize** follow the pointer without a button held: Move
keeps the window where the pointer took it, Resize moves the window's
bottom-right corner. A click ends it; Escape puts the window back as it
was. The keyboard works too: the arrow keys move the pointer (so the
window, or its corner) by 10 logical pixels, 1 with Control held;
Return, keypad Enter or Space ends it as a click does. Meanwhile key
presses do not reach apps.

Keys from `org.gnome.desktop.wm.keybindings` acting on the focused window:

| Key | Effect |
|-----|--------|
| `close` | close it, as its own close button would |
| `minimize` | minimize |
| `maximize`, `unmaximize`, `toggle-maximized` | maximize, restore, or switch |
| `maximize-horizontally`, `maximize-vertically` | fill the work area in that direction (again: restore) |
| `toggle-fullscreen` | fullscreen or back |
| `toggle-above`, `always-on-top` | switch Always on Top |
| `begin-move`, `begin-resize` | Move / Resize as from the menu |
| `move-to-monitor-left` / `-right` / `-up` / `-down` | to the nearest monitor that way, keeping its place relative to the work area (maximized and fullscreen windows stay so) |
| `move-to-center` | center it in its work area |
| `move-to-side-n` / `-s` / `-e` / `-w`, `move-to-corner-ne` / `-nw` / `-se` / `-sw` | against that edge or corner of the work area |
| `raise`, `lower`, `raise-or-lower` | above all windows, below all, or whichever it is not (lowering focuses the new top window) |
| `show-desktop` | minimize every window on the workspace; again (before any is shown) shows them back |

## Tiling (Helix)

`org.gnome.mutter.keybindings` `toggle-tiled-left` and `toggle-tiled-right`
(defaults `<Super>Left`, `<Super>Right`) tile the focused window: it fills
the left or right half of its monitor's work area (its title bar, if
Helix draws one, included). The same key again restores the window's
earlier size and place; the other key moves it to the other half. Like a
maximized window, a tiled window follows its work area when that changes,
restoring it returns it to where it was before, and dragging it restores
it once the pointer has moved. Wayland apps are told which edges are tiled
(`xdg_toplevel` states `tiled_left`/`right`/`top`/`bottom`); tiled windows
are not reported as maximized.

With `org.gnome.mutter edge-tiling` true, dragging a window (by its title
bar, its own drag area, or `<Super>`+drag) until the pointer touches the
left or right edge of a monitor, where no other monitor continues,
previews the half it would fill (a translucent rectangle); releasing there
tiles it. Touching the top edge previews and then maximizes. Moving away
from the edge drops the preview.

`switch-monitor`, `rotate-monitor` and `cancel-input-capture` are not
handled.

## Window-management settings (Helix)

Helix follows these keys of `org.gnome.desktop.wm.preferences` and
`org.gnome.mutter`; changes apply at once.

**Focus.** `focus-mode` `click` (default): clicking a window focuses it.
`sloppy`: the window the pointer moves into gets focus (moving over the
desktop or the panel changes nothing). `mouse`: as `sloppy`, and moving
the pointer onto the desktop also takes focus away. With
`focus-change-on-pointer-rest` (in `sloppy`/`mouse`), focus changes only
once the pointer has rested for 150 ms. Focus by pointer does not raise,
unless `auto-raise` is on: then the window is raised after
`auto-raise-delay` milliseconds if it still has focus. With
`raise-on-click` false, clicking a window's own area focuses it without
raising it; clicks on title bars Helix draws, and modifier clicks, still
raise.

**Modifier clicks.** Holding `mouse-button-modifier` (default `<Super>`;
empty or unparsable: off): a left drag moves the window, a middle drag
resizes it, a right click opens the window menu; with
`resize-with-right-button` the right drag resizes and the middle click
opens the menu. A resize moves the corner nearest the pointer (the window
is split into quarters). These clicks are not passed to the app.

**New windows.** They open on the monitor under the pointer, cascading
from the top-left corner of its work area, or centered in it with
`center-new-windows`. With `auto-maximize`, a new window at least as
large as the work area (title bar included) opens maximized. Dialogs
(windows with a parent) open centered over their parent; with
`attach-modal-dialogs`, a modal dialog (`xdg_dialog_v1` modal) opens
centered at its parent's top edge and moves with its parent.
`focus-new-windows` `smart` (default): new windows get focus and go on
top; `strict`: a new window goes just below the focused window without
taking focus (it does get focus when no window has it).

**Overlay key.** `overlay-key` (default `Super`, meaning either Super
key; otherwise a key name such as `Super_R`; empty: none) pressed and
released alone shows the application launcher.

**Bells.** An app rings the bell with `xdg_system_bell_v1`. With
`audible-bell` the `bell` sound of the sound theme plays (subject to
`org.gnome.desktop.sound event-sounds`). With `visual-bell`,
`visual-bell-type` `fullscreen-flash` flashes the monitor of the window
(or of the pointer) white for a moment; `frame-flash` flashes the
window itself (the focused one when the app is unknown).

**Locate pointer.** With `org.gnome.desktop.interface locate-pointer`,
pressing and releasing `locate-pointer-key` (default `Control_L`) alone
shows shrinking circles around the pointer for half a second.

## Unresponsive apps (Helix)

When a Wayland app's window gets focus, Helix checks that the app still
responds (`xdg_wm_base.ping`). Without an answer within
`org.gnome.mutter check-alive-timeout` milliseconds (default 5000; 0 turns
checking off), the shell shows a dialog: "“TITLE” is not responding.",
"You may choose to wait a short while for it to continue or force the app
to quit entirely.", with **Wait** and **Force Quit**.

- **Wait** closes the dialog; if the app still has not answered after
  another `check-alive-timeout`, the dialog comes back.
- **Force Quit** ends the app: its process (the Wayland client's peer
  process) is killed with `SIGKILL`.
- An answer from the app closes the dialog.

X11 apps are not checked.

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
- In the application switcher, Down or the key above Tab shows the
  selected application's windows (the switcher becomes a window switcher
  for that application, its most recent window selected); Up goes back
  to the applications.
- `cycle-windows` (default Alt+Escape) and `cycle-group` (Alt+F6) switch
  directly, without a switcher: each press activates the next window, of
  every application or of the focused window's application, in most
  recently used order (backward with the `-backward` keys). A cycle starts
  with that order at the first press; presses less than 1.5 seconds apart
  continue along it (as the shell does not see the modifier being let go
  without a switcher), and a pause starts a new cycle. They follow
  `current-workspace-only` of `org.gnome.shell.window-switcher`.
- `switch-panels` (Ctrl+Alt+Tab) moves keyboard focus to the panel: Left,
  Right and Tab (and the key again) move between its controls (Apps,
  Power), Enter or Space activates one, Escape leaves.
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
  workspace, or all workspaces) and the workspaces (how many, which is active), and
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

## Input methods (Helix)

The compositor offers `zwp_text_input_v3` to every client, and
`zwp_input_method_v2` and `zwp_virtual_keyboard_v1` only to the shell it
launched. The shell is the session's input method (for `ibus` input
sources, `contracts/gsettings/input-settings.md`): while it grabs the
keyboard, keys go to it, and it sends the app's text field committed and
pre-edit text, and the keys it does not use through its virtual keyboard.
Its candidate popups are placed next to the text cursor the app reports.

## Global keys (Helix)

The shell binds its keys through Helix's private `helix_keybindings_v1`
protocol. Bound keys do nothing while the session is locked, except
those the shell marks as working while locked (volume and brightness).
