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

**Window icons** (`xdg_toplevel_icon_manager_v1`): see "Window
switching" for where they show.

**Xwayland options** (`org.gnome.mutter.wayland`, read when Xwayland
starts, so changes take effect in the next session):
`xwayland-disable-extension` `Security` and `Xtest` disable the X
`SECURITY` and `XTEST` extensions, when this Xwayland can switch them off
(otherwise the flag is ignored); `xwayland-allow-byte-swapped-clients`
lets X11 clients of the other byte order connect (Xwayland refuses them
otherwise).

**X11 apps: scaling and settings.** Xwayland's coordinate space is
scaled by `org.gnome.mutter.wayland xwayland-scaling-factor`, rounded to a
whole number between 1 and 4; 0 (the default) means automatic, which in
Helix is the largest monitor scale rounded to a whole number (1.25 gives 1,
1.5 and 1.75 give 2). At scale *n* the X screen is *n* times the size of
the monitors' logical layout: X11 apps draw *n* times larger, and Helix
shows their windows *n* times smaller (Wayland apps are unaffected). The
scale follows the setting and monitor changes while the session runs;
X11 apps that cannot change scale while running look right after they are
restarted. Apps that do not scale themselves look smaller at a scale
above 1.

Helix is the XSETTINGS manager of Xwayland's screen (it owns
`_XSETTINGS_S0`, announces itself with a `MANAGER` client message, and
keeps the `_XSETTINGS_SETTINGS` property of its window, per the
freedesktop XSETTINGS specification, little-endian, the serial raised on
each change). It publishes, from GSettings:

| Setting | Value |
|---------|-------|
| `Gdk/WindowScalingFactor` | the Xwayland scale |
| `Gdk/UnscaledDPI` | 96 × `text-scaling-factor` × 1024 |
| `Xft/DPI` | 96 × `text-scaling-factor` × scale × 1024 |
| `Xft/Antialias`, `Xft/Hinting` | 1 unless `font-antialiasing` / `font-hinting` is `none` |
| `Xft/HintStyle` | `hint` + `font-hinting` (`hintslight`, …) |
| `Xft/RGBA` | `font-rgba-order` when `font-antialiasing` is `rgba`, else `none` |
| `Net/ThemeName`, `Net/IconThemeName`, `Gtk/FontName`, `Gtk/KeyThemeName` | `gtk-theme`, `icon-theme`, `font-name`, `gtk-key-theme` |
| `Gtk/CursorThemeName`, `Gtk/CursorThemeSize` | `cursor-theme`, `cursor-size` × scale |
| `Net/CursorBlink`, `Net/CursorBlinkTime`, `Gtk/CursorBlinkTimeout` | `cursor-blink`, `cursor-blink-time`, `cursor-blink-timeout` |
| `Gtk/EnableAnimations`, `Gtk/OverlayScrolling` | `enable-animations`, `overlay-scrolling` |
| `Gtk/DecorationLayout` | `org.gnome.desktop.wm.preferences button-layout` |
| `Net/EnableEventSounds`, `Net/SoundThemeName` | `org.gnome.desktop.sound event-sounds`, `theme-name` |
| `Net/DoubleClickTime` | `org.gnome.desktop.peripherals.mouse double-click` |

(keys without a schema are in `org.gnome.desktop.interface`;
`text-scaling-factor` is taken between 0.5 and 3). The root window's
`RESOURCE_MANAGER` gets `Xft.dpi` (96 × `text-scaling-factor` × scale),
`Xft.antialias`, `Xft.hinting`, `Xft.hintstyle`, `Xft.rgba`,
`Xcursor.theme` and `Xcursor.size` (`cursor-size` × scale), replacing
earlier lines of those names and keeping others. Both follow setting
changes. If another XSETTINGS manager already owns the selection, Helix
publishes nothing.

**Xwayland's display.** Helix lets Smithay pick the first free display
from `:0` to `:32`; when all are taken it tries `:33` to `:99` in turn.

**Foreign toplevels** (`zxdg_exporter_v2`/`zxdg_importer_v2`, and v1). An
app (typically a portal) may make its toplevel a dialog of another app's
exported toplevel: it is then placed and kept as dialogs are (see
"Window-management settings").

**Gestures.** Some swipes are Helix's own and are not passed to apps.
A finished swipe does one of:

- **sideways** (at least 100 logical pixels, and more sideways than up
  or down): switch workspace the way the content was pulled (fingers
  moving left show the next workspace, to the right; moving right the
  previous one), as GNOME does regardless of `natural-scroll`; there is
  none past the first or last workspace;
- **up** (at least 100 logical pixels, more up than sideways): open the
  overview; when it is already open, show its application grid
  (`org.gnome.Shell` `OverviewActive`, `ShowApplications`);
- **down**: close the overview when it is open.

A cancelled swipe, or one while the session is locked, does nothing.

- **Touchpads** (`zwp_pointer_gestures_v1`, bare hardware): swipes with 3
  or 4 fingers are Helix's, measured by libinput's swipe motion. Other
  gestures (two-finger swipes, pinches, holds) go to the surface under
  the pointer.
- **Touchscreens** (and remote desktop virtual touchscreens): when a
  third finger touches while two are down, the touch points become
  Helix's (apps get `wl_touch.cancel`; further touches go to Helix until
  every finger has lifted); the swipe is the average movement of the
  fingers from where each went down to where it lifted.
- **Edge swipes** (touchscreens): a single finger that goes down within
  16 logical pixels of an edge of the monitor it touches (the nearest
  edge in a corner) and moves at least 48 logical pixels inwards, more
  inwards than along the edge, becomes Helix's (apps get
  `wl_touch.cancel`); when it lifts: from the bottom edge, **up**; from
  the top edge, **down**; from the left edge, the previous workspace;
  from the right edge, the next. A second finger before that leaves the
  touch to the app.
- Not yet tested with a real touchscreen or touchpad: the recognizer is
  checked by unit tests, and touchscreen swipes in a nested session with
  a remote desktop virtual touchscreen (`gaps.md`).

**Frame timing.** `wp_presentation` feedback reports when a frame was
shown (the monitor's vblank, its refresh interval and sequence) or that
it was discarded. `wp_fifo_manager_v1`: a commit waiting on a FIFO barrier
is applied after the next frame of the surface's monitor.
`wp_commit_timing_manager_v1`: a commit with a target time is applied at
the first frame at or after it. Commits of windows not shown (minimized,
on another workspace) are not held back. `wp_single_pixel_buffer_manager_v1`
is offered.

**Drawing only what changed.** On bare hardware and nested, a monitor is
drawn only when something on it changed since the buffer being drawn was
last shown; when nothing did, Helix looks again about one refresh later
(frame callbacks still go out), so an idle desktop costs next to nothing.

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
- Every surface's preferred description, and a monitor's in color mode
  0, is sRGB: `srgb` primaries (red 0.64/0.33, green 0.30/0.60, blue
  0.15/0.06, white 0.3127/0.3290), the `srgb` transfer function,
  luminances 0.2 / 80 / 80 cd/m² (minimum / maximum / reference white)
  and the same target volume. `preferred_changed` is never sent.
- A monitor in the BT.2100 color mode (`org.gnome.Mutter.DisplayConfig.md`,
  "Color modes") is described as `bt2020` primaries (red 0.708/0.292,
  green 0.170/0.797, blue 0.131/0.046, white 0.3127/0.3290), the
  `st2084_pq` transfer function, and as luminances its minimum and
  maximum (from its EDID, else 0.005 and 1000 cd/m²) and the reference
  white, with the same target volume. When a monitor's color mode
  changes, its `wp_color_management_output_v1` objects get
  `image_description_changed`. A monitor that has gone away gives a
  description that fails with `no_output`.
- Content is still taken as sRGB: apps cannot describe their surfaces as
  BT.2100 (the transfer function and primaries are not advertised); on a
  BT.2100 monitor the compositor converts everything from sRGB.

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
released alone shows the overview, or hides it (see "Overview").

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

When a window gets focus, Helix checks that its app still responds: a
Wayland app with `xdg_wm_base.ping`, an X11 app with `_NET_WM_PING` (only
windows listing it in `WM_PROTOCOLS`; the app answers by sending the
message back to the root window). Without an answer within
`org.gnome.mutter check-alive-timeout` milliseconds (default 5000; 0 turns
checking off), the shell shows a dialog: "“TITLE” is not responding.",
"You may choose to wait a short while for it to continue or force the app
to quit entirely.", with **Wait** and **Force Quit**.

- **Wait** closes the dialog; if the app still has not answered after
  another `check-alive-timeout`, the dialog comes back.
- **Force Quit** ends the app: a Wayland app's process (the client's peer
  process) is killed with `SIGKILL`; an X11 app's connection to the display
  is closed (`XKillClient`).
- An answer from the app closes the dialog.

## Overview (Helix)

The overview shows the open windows of the active workspace and all the
workspaces. The primary monitor holds all of it; every other monitor
shows the active workspace's windows that are on it (by their middle),
laid out the same way. It is shown by the overlay key, by
`org.gnome.shell.keybindings toggle-overview`, and by setting
`org.gnome.Shell` `OverviewActive`; the same key again, Escape, or setting
`OverviewActive` false hides it. It holds, from the top:

- **A search field.** Typing opens the application launcher with what was
  typed as its search (see "Window switching" for the launcher's window
  results).
- **Workspace thumbnails**, one per workspace in order: a small picture of
  the primary monitor with the windows on that workspace where they are
  (minimized windows left out; windows shown on every workspace in all of
  them), the active workspace outlined. Clicking one switches to it; the
  overview stays open and shows that workspace's windows.
- **The active workspace's windows** (minimized ones and those on every
  workspace included) of this monitor, as pictures renewed about twice a
  second while the overview is open, in rows, each keeping its proportions and at most
  its real size, its title below. The focused window is selected; the
  arrow keys move the selection and Enter activates it. Clicking a window
  activates it and hides the overview. Its × button (shown on the picture
  under the pointer and on the selected one) closes the window, as its
  close button would. Dragging a window onto a workspace thumbnail moves it
  to that workspace.

**Hot corner.** While `org.gnome.desktop.interface` `enable-hot-corners`
is true, moving the pointer into the top-left corner of the primary
monitor (the corner pixel) shows the overview, or hides it when it (or the
launcher) is shown. The corner works again once the pointer has left the
32×32 pixel square at the corner. It does nothing while a pointer button
is held, while the session is locked, while a fullscreen window is on the
primary monitor, while the screenshot UI is shown, or when another monitor lies to the left of or above the
corner (it is then no screen corner).

The launcher (the Apps button, `toggle-application-view`,
`org.gnome.Shell` `ShowApplications`, `FocusSearch` and `FocusApp`) and the
overview each replace the other. `OverviewActive` is true while either is
shown.

## Favorites and app folders (Helix)

**Favorites** are `org.gnome.shell` `favorite-apps` (desktop file ids, in
order; ids of apps not installed, or hidden from menus, are skipped).

- **Dash.** The overview shows, centered at the bottom of the primary
  monitor, the favorites' icons, then (after a gap) the icons of other
  apps that have windows, most recently used first; a dot under an icon
  means the app has windows. Clicking an icon activates the app's most
  recently used window, or launches the app when it has none, and hides
  the overview.
- **Switching to apps.** `org.gnome.shell.keybindings
  switch-to-application-1` … `-9` (default Super+1 … Super+9) act on the
  first to ninth app of the dash (favorites, then other running apps), as
  a click on the dock does with `cycle-windows`: the app's newest window
  is activated, the next of its windows when one of them has focus, and an
  app without windows is launched.
- **Launcher.** With an empty search the launcher lists the favorites
  first (tagged "Favorite"), then the app folders, then the other apps
  that are in no folder, by name. Ctrl+D adds the selected app to the
  end of the favorites, or removes it.

## Dock (Helix)

Besides the overview's dash, Helix shows a dock on the desktop, like
Ubuntu's Dash to Dock. It follows that extension's settings,
`org.gnome.shell.extensions.dash-to-dock` (key table in
`contracts/gsettings/org.gnome.shell.extensions.dash-to-dock.md`), when
the schema is installed, so Ubuntu's Settings controls it; without the
schema it uses the defaults below. It is off while `org.gnome.shell
disabled-extensions` lists `ubuntu-dock@ubuntu.com` or
`dash-to-dock@micxgx.gmail.com`, and it is not shown on the lock screen,
in the overview (which has its dash) or the launcher.

- **Where.** With `multi-monitor`, on every monitor (each its own dock,
  shown and hidden on its own); otherwise on one monitor: the one whose
  connector is `preferred-monitor-by-connector`, else the primary monitor
  (`primary`, the default). On the edge `dock-position` names (`BOTTOM` by default;
  `TOP` is below the panel). Centered on the edge, or along all of it with
  `extend-height` (then the icons start at the edge's beginning unless
  `always-center-icons`). `height-fraction` is not used.
- **What.** In order: the Show Apps button when `show-apps-at-top`; the
  favorites (`show-favorites`); apps with windows that are not favorites,
  in the order their first window opened (`show-running`; with
  `isolate-workspaces`, only windows of the current workspace count; with
  `isolate-monitors`, only windows whose middle is on the dock's monitor,
  which also goes for the running marks and clicks);
  mounted drives that can be unmounted or ejected (`show-mounts`:
  removable drives, discs, phones, network shares); the Trash
  (`show-trash`, its icon full or empty); the Show Apps button
  (`show-show-apps-button`) unless at the top. A thin line separates
  these groups.
- **Looks.** Icons are `dash-max-icon-size` (default 48) pixels, made
  smaller (to 16) when they do not fit the edge, unless
  `icon-size-fixed`. The bar is dark, with opacity `background-opacity`
  for `transparency-mode` `FIXED`, 0.8 for `DEFAULT`, and for `DYNAMIC`
  `background-opacity` or opaque while a window touches the dock. Under
  (or beside) each running app, one mark per window (at most four):
  `running-indicator-style` `DOTS` / `DEFAULT` dots, `SQUARES` / `BINARY`
  squares, `DASHES` / `SEGMENTED` dashes, `SOLID` / `METRO` one bar; the
  other styles are dots. The focused app's icon is highlighted. With
  `show-icons-notifications-counter`, an app with notifications in the
  list has a red badge with their number (9+ above nine). Pointing at an
  item shows its name (the app's, the drive's, "Trash", "Show Apps")
  beside the bar.
- **Showing and hiding.** With `dock-fixed` the dock is always shown and
  keeps its room (windows are not placed under it). Otherwise it is above
  windows, and with `intellihide` it hides while a window of the current
  workspace overlaps it, which windows counting by `intellihide-mode`:
  `ALL_WINDOWS`, `FOCUS_APPLICATION_WINDOWS` (the focused app's, the
  default), `MAXIMIZED_WINDOWS` (those spanning the monitor along the
  dock's edge) or `ALWAYS_ON_TOP` (none). With `autohide` and without
  `intellihide` it is hidden unless the pointer is on it. A hidden dock
  with `autohide` shows again when the pointer reaches the screen edge
  (after `show-delay` seconds) and hides `hide-delay` seconds after the
  pointer leaves it (1.5 seconds after showing if the pointer never comes
  onto it). While the focused window covers the whole monitor (full
  screen) the dock is hidden, and shows at the edge only with
  `autohide-in-fullscreen`. Helix triggers at the edge pixel, without
  pressure (`require-pressure-to-show`, `pressure-threshold` are not
  used), and does not animate (`animation-time`).
- **Clicking an app.** `click-action` (left button) and
  `middle-click-action` decide (`shift-click-action` is not used: the
  shell does not see modifiers on dock clicks): `cycle-windows` activates
  the app's newest window, or when one of its windows has focus the next
  of its windows (by the order they opened); `minimize` minimizes the
  app's windows when one has focus, else activates; `launch` starts it
  again; `minimize-or-overview` minimizes when focused, opens the
  overview when it has several windows, else activates;
  `focus-or-appspread` likewise without minimizing; `previews`,
  `focus-or-previews`, `minimize-or-previews` and
  `focus-minimize-or-previews` show a menu of its windows when it has
  several (Helix has no window previews); `focus-minimize-or-appspread`
  minimizes when focused, else opens the overview or activates; `quit`
  closes its windows; `skip` does nothing. An app without windows is
  launched (except with `skip`). A drive opens in the file manager, the
  Trash opens, Show Apps opens or closes the launcher.
- **Scrolling** over the dock, by `scroll-action`: `cycle-windows`
  activates the next (or previous) window of the app under the pointer;
  `switch-workspace` switches to the workspace to the right (or left).
- **Right click** opens a menu. For an app: its windows by title (each
  activates it), then "Open" (no windows) or "New Window", the desktop
  file's actions (`Actions=`), then "Pin to Dash" or "Unpin" (adds it to
  or removes it from `favorite-apps`), "App Details" (GNOME Software's
  page for it, when `gnome-software` is installed), and for a running app
  "Quit" or "Quit <n> Windows" (closes its windows). For a drive: "Open",
  "Eject" (or "Unmount"). For the Trash: "Open" and, when it holds
  anything, "Empty Trash", which asks first ("Empty Trash?", "All items in
  the Trash will be permanently deleted.", "Cancel" / "Empty Trash") and
  then empties the trash of every mount (as `gio trash --empty` does).
- **Dragging** an app's icon along the dock moves it among the favorites
  (a running app dropped there becomes a favorite at that place);
  dropping a favorite more than 60 pixels away from the dock unpins it.
- Apps' counts, progress and urgency from
  `com.canonical.Unity.LauncherEntry` show on their icons
  (`contracts/dbus/com.canonical.Unity.LauncherEntry.md`); an app's count
  replaces its notification count. Urgent apps get an orange background
  (`dance-urgent-applications` animation is not done).
- `hot-keys`, `app-hotkey-*`, the window previews and the remaining keys
  are not used.

**App folders** are `org.gnome.desktop.app-folders` `folder-children`; each
child `<id>` is a `org.gnome.desktop.app-folders.folder` at
`/org/gnome/desktop/app-folders/folders/<id>/`:

- its name is `name`, or with `translate` true the `Name` of the desktop
  directory entry `name` (in `desktop-directories/` under the XDG data
  directories), else `name` as it is;
- an app is in it when its id is in `apps`, or one of its desktop file's
  `Categories` is in `categories`, unless its id is in `excluded-apps`;
- folders with no installed app are left out.

A folder's row (tagged "Folder") opens it: the launcher then lists the
folder's apps, its name in the search field's place. Backspace with an
empty search, or Escape, goes back; typing searches all apps.

## Accessibility menu (Helix)

While `org.gnome.desktop.a11y always-show-universal-access-status` is
true, or any of its switches is on, the panel shows an accessibility
indicator (a figure with arms out) left of the input source indicator.
Clicking it opens a menu below it with a check box per switch, in this
order (a switch whose key is not installed is left out):

| Switch | Setting |
|--------|---------|
| High Contrast | `org.gnome.desktop.a11y.interface high-contrast` |
| Zoom | `org.gnome.desktop.a11y.applications screen-magnifier-enabled` |
| Large Text | `org.gnome.desktop.interface text-scaling-factor`: on is 1.25, off 1.0; shown on when above 1.0 |
| Screen Reader | `org.gnome.desktop.a11y.applications screen-reader-enabled` |
| Screen Keyboard | `org.gnome.desktop.a11y.applications screen-keyboard-enabled` |
| Visual Alerts | `org.gnome.desktop.wm.preferences visual-bell` |
| Sticky Keys | `org.gnome.desktop.a11y.keyboard stickykeys-enable` |
| Slow Keys | `org.gnome.desktop.a11y.keyboard slowkeys-enable` |
| Bounce Keys | `org.gnome.desktop.a11y.keyboard bouncekeys-enable` |
| Mouse Keys | `org.gnome.desktop.a11y.keyboard mousekeys-enable` |

Choosing a switch turns it on or off and closes the menu. The menu
follows changes made elsewhere (it shows the settings as they are when
opened); the indicator comes and goes with them.

## Quick settings (Helix)

The panel's **Power** button and `org.gnome.shell.keybindings
toggle-quick-settings` (default Super+S) open or close a menu under the
panel's right end, holding, from the top:

- **Volume**: a slider for the default output's volume (0 to 100 %, or to
  150 % with `org.gnome.desktop.sound allow-volume-above-100-percent`) and
  the level in percent; clicking the slider sets the volume where it was
  clicked (and unmutes); clicking "Volume" mutes or unmutes. Without a
  sound server (`pactl`) the row is left out.
- **Brightness**, when a monitor's backlight can be controlled
  (DisplayConfig `Backlight`): a slider setting every such monitor to the
  same fraction of its range.
- Switches, highlighted while on, in rows of three: **Wi-Fi** (when
  NetworkManager runs; its label is the connected network's name while
  connected), **Bluetooth** (when BlueZ has an adapter), **Airplane
  Mode** (when either is there), **Auto Rotate** (when iio-sensor-proxy
  has an accelerometer: `orientation-lock`, inverted), **Night Light**
  (`org.gnome.settings-daemon.plugins.color night-light-enabled`), **Dark
  Style** (`org.gnome.desktop.interface color-scheme` `prefer-dark`, else
  `default`) and **Do Not Disturb** (`org.gnome.desktop.notifications
  show-banners`, inverted). The Wi-Fi and Bluetooth switches have a "›"
  at their right end that opens (or closes) their list under the
  switches; only one list is open at a time.
- **Power Mode**, when power-profiles-daemon runs: one button per profile
  it offers (Power Saver, Balanced, Performance), the active one
  highlighted; clicking one makes it active.
- The power actions logind allows: Lock Screen, Suspend, Log Out…,
  Restart…, Power Off….

Escape or a click outside closes it; choosing a power action closes it
too. The values are read when it opens and after each change.

The switches act on (all on the system bus):

- Wi-Fi: NetworkManager's `WirelessEnabled` property
  (`org.freedesktop.NetworkManager`).
- Bluetooth: `Powered` of every BlueZ adapter (`org.bluez.Adapter1`).
- Airplane Mode is on while Wi-Fi is off (or absent) and no adapter is
  powered (or there is none). Turning it on turns Wi-Fi, mobile broadband
  (`WwanEnabled`) and Bluetooth off; turning it off turns Wi-Fi and
  Bluetooth on.
- Power Mode: power-profiles-daemon's `ActiveProfile`
  (`org.freedesktop.UPower.PowerProfiles` at
  `/org/freedesktop/UPower/PowerProfiles`, else the older
  `net.hadess.PowerProfiles`), from its `Profiles`.

### Wi-Fi networks and Bluetooth devices

- **Networks** (while Wi-Fi is on): the access points of NetworkManager's
  Wi-Fi devices (`DeviceType` 2; `org.freedesktop.NetworkManager.Device.Wireless`
  `AccessPoints`), one row per network name (`Ssid`; hidden networks
  without one are left out), keeping the strongest (`Strength`), strongest
  first, at most eight. Each row shows the signal icon (as in the status
  area), the name, a lock when the network is secured (`Flags` privacy bit
  or non-zero `WpaFlags`/`RsnFlags`) and "Connected" for the device's
  `ActiveAccessPoint`.
- Choosing a network that is not connected: when a saved connection is
  for that name (`org.freedesktop.NetworkManager.Settings`
  `ListConnections`, each connection's `GetSettings` `802-11-wireless`
  `ssid`), `ActivateConnection(connection, device, access point)`; else,
  for an open network, `AddAndActivateConnection({}, device, access
  point)`; else, for a new secured network, also
  `AddAndActivateConnection({}, device, access point)`: NetworkManager
  then asks the shell's secret agent for the password
  (`contracts/dbus/org.freedesktop.NetworkManager.SecretAgent.md`).
  Choosing the connected network does nothing.
- **Devices** (while Bluetooth is on): BlueZ's paired devices
  (`org.freedesktop.DBus.ObjectManager` of `org.bluez` at `/`, objects with
  `org.bluez.Device1` `Paired` true), by `Alias`, at most eight, with
  "Connected" for connected ones. Choosing one calls `Connect()` or, when
  connected, `Disconnect()`.
- **Send Files…**, below the devices while any is paired: closes the menu
  and sends files over Bluetooth, asking for the device and the files
  (`contracts/dbus/org.bluez.obex.md`, "Sending").
- **Pair New Device…**, the last row of the device list: starts discovery
  on every powered adapter (`org.bluez.Adapter1` `StartDiscovery()`); the
  list then shows the unpaired devices found that have a name, at most
  eight, under the heading "Nearby Devices". Choosing one calls
  `Pair()` (codes go through the shell's pairing agent,
  `contracts/dbus/org.bluez.Agent1.md`), then `Trusted` true and
  `Connect()`; when pairing fails a notification "Cannot pair with
  <name>" gives the reason. Closing the list or the menu stops discovery
  (`StopDiscovery()`).
- The lists are read with the rest of the status (see "System status");
  "No Networks" or "No Devices" when empty.

## System status (Helix)

Left of the Power button the panel shows, right to left:

- **Battery**, when UPower's display device
  (`/org/freedesktop/UPower/devices/DisplayDevice`) is present: its
  `IconName`, and with `org.gnome.desktop.interface
  show-battery-percentage` its `Percentage`.
- **Volume**: `audio-volume-muted-symbolic` (muted or at 0),
  `-low-` (to 33 %), `-medium-` (to 66 %) or `-high-symbolic`, when a
  sound server answers.
- **Network**, when NetworkManager runs: for the primary connection,
  `network-wireless-signal-{none,weak,ok,good,excellent}-symbolic` (Wi-Fi,
  by the access point's strength: below 20, 40, 50, 80 and above),
  `network-wired-symbolic` (Ethernet), `network-vpn-symbolic` (VPN), or
  `network-cellular-connected-symbolic` (mobile); without one,
  `network-wireless-offline-symbolic` while Wi-Fi is on, else
  `network-offline-symbolic`; `airplane-mode-symbolic` in airplane mode.
- **Location**: `location-services-active-symbolic` while an app uses
  GeoClue (`contracts/dbus/org.freedesktop.GeoClue2.Agent.md`, "Location
  in use").
- **Microphone**: `audio-input-microphone-symbolic` while a PipeWire node
  of `media.class` `Stream/Input/Audio` is running, except streams that
  record an output rather than an input (`stream.capture.sink` true) or
  only watch levels (`stream.monitor` true).
- **Camera**: `camera-web-symbolic` while a camera is running: a PipeWire
  node of `media.class` `Video/Source` whose `media.role` is `Camera` or
  whose `device.api` is `v4l2` or `libcamera` (screen casts are not
  cameras).

Icons come from the icon theme (`org.gnome.desktop.interface
icon-theme`, then `Adwaita`, then `hicolor`), drawn in the panel's text
color. The status follows the services' property changes, and is read
again every 5 seconds; clicking it opens the quick settings.

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
  ignored: Helix shows no thumbnails, and every entry has an icon.
- Switcher entries show a window's icon: the one its app set
  (`xdg_toplevel_icon_manager_v1`: an icon theme name, or the largest
  image it gave; for X11 windows the largest square image in
  `_NET_WM_ICON`, else its largest image centered on a transparent
  square, followed as it changes), else its app's icon (the icon of the desktop file named
  by its app id, or for X11 windows its `WM_CLASS` class). Helix asks for
  24, 48 and 96 pixel images (`icon_size`).
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
