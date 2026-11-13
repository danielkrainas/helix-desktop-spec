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
true, an X11 window may grab the keyboard when its `WM_CLASS` name or
class matches an entry of `xwayland-grab-access-rules` or of Helix's
built-in list, or when it asked for it, and no `!` entry of the rules
denies it (`*` and `?` wildcards; `!` entries win over everything). It
then gets all keys, as an allowed shortcut inhibitor does, until it ends
the grab or `restore-shortcuts` is pressed.

- The built-in list: `Xephyr`, `Xnest`, `VirtualBox Machine`,
  `VirtualBoxVM`, `Vncviewer`, `vncviewer`, `remote-viewer`, `xfreerdp`,
  `rdesktop`, `qemu*` (nested X servers, virtual machine and remote
  desktop viewers).
- Asking: an X11 client sends a `ClientMessage` of type
  `_XWAYLAND_MAY_GRAB_KEYBOARD` for its window to the root window
  (substructure masks), `data32[0]` 1 to ask and 0 to stop asking. The
  compositor's own Xwayland connection receives it; the answer lasts for
  that window id.

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
| `Net/DndDragThreshold` | `org.gnome.desktop.peripherals.mouse drag-threshold` × scale |
| `Net/EnableInputFeedbackSounds` | `org.gnome.desktop.sound input-feedback-sounds` (1 or 0) |
| `Gtk/RecentFilesEnabled`, `Gtk/RecentFilesMaxAge` | `org.gnome.desktop.privacy remember-recent-files` (1 or 0), `recent-files-max-age` |

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
(`wp_cursor_shape_manager_v1`, version 2: all its shapes; for the pointer
taken while one of the app's surfaces has the pointer, or started the
pointer grab, the request's serial not compared; for tablet tools see
`contracts/gsettings/input-settings.md`; an unknown shape is the
protocol's `invalid_shape` error), or nothing. Elsewhere (desktop, panels
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

Helix offers `wp_color_manager_v1` (version 1). It composites in its own
space: BT.709 (sRGB) primaries, gamma 2.2 encoding, reference white at
the full signal; surfaces without an image description are taken to be
in it. Surfaces whose image description says otherwise are converted
into it as they are drawn; in the BT.2100 color mode the whole result
is then converted for the monitor ("Color modes" in
`org.gnome.Mutter.DisplayConfig.md`).

- Advertised: the perceptual and relative render intents (drawn the
  same); the features `parametric`, `set_primaries`, `set_tf_power`,
  `set_luminances`, `set_mastering_display_primaries` and
  `windows_scrgb`; the transfer functions `bt1886`, `gamma22`, `gamma28`,
  `ext_linear`, `srgb`, `ext_srgb` and `st2084_pq`; the primaries
  `srgb`, `pal_m`, `pal`, `ntsc`, `generic_film`, `bt2020`, `dci_p3`,
  `display_p3` and `adobe_rgb`. ICC profiles (`create_icc_creator` is
  the `unsupported_feature` error), HLG and the logarithmic and other
  transfer functions, and `extended_target_volume` are not offered; other
  named values are the creator's `invalid_tf` / `invalid_primaries_named`
  errors.
- Parametric descriptions follow the protocol: the transfer function and
  primaries are required (`incomplete_set`), each property is set once
  (`already_set`), `set_tf_power` takes exponents from 1.0 to 10.0,
  luminances must lie above the minimum, and `max_cll` / `max_fall` must
  fit each other and the mastering luminance range
  (`invalid_luminance`). Unset luminances take the transfer function's
  defaults (`st2084_pq`: 0.005 / 10000 / 203 cd/m²; others 0.2 / 80 /
  80); with `st2084_pq` the maximum is the minimum plus 10000 cd/m². A
  created description is ready at once; equal descriptions share an
  identity. Descriptions from creators and `create_windows_scrgb` give
  no information (`no_information`); those of monitors and surface
  feedback do: primaries (and their name when named), transfer function,
  luminances, target primaries and luminance (the description's own
  when no mastering values were set), and `max_cll` / `max_fall` when
  set.
- Windows-scRGB is `srgb` primaries with `ext_linear`; Helix takes 1.0
  (80 cd/m²) as reference white, as EGL's scRGB-linear color space does
  (the protocol leaves the reference unknown).
- A surface's description (`set_image_description`, any ready one with a
  render intent offered; `unset_image_description`; destroying the
  object unsets it) takes effect on its next commit. A second
  `wp_color_management_surface_v1` for a surface is `surface_exists`;
  once the surface is gone the object is inert (`inert`).
- **Conversion.** A window drawn with a surface whose description is not
  already the composition space (BT.709 primaries with the `srgb`,
  `ext_srgb` or `gamma22` transfer function and reference white at the
  maximum) is drawn from its surfaces' textures, those surfaces through a
  shader: the transfer function is decoded (`st2084_pq` to cd/m², `bt1886`
  as a 2.4 power, `ext_linear` as is, `srgb` piecewise, a power as given;
  negative values keep their sign), scaled so the description's
  reference white is 1.0, converted to BT.709 (Bradford adaptation when
  the white points differ), clipped to 0–1 (no tone mapping: anything
  brighter than reference white is clipped, also on BT.2100 monitors),
  and encoded with gamma 2.2. Such windows fade but do not grow or shrink
  as they open and close, and are drawn again in full each frame. Shell
  surfaces (layer surfaces) are not converted.
- Monitors: in color mode 0 a monitor's description is sRGB (`srgb`
  primaries — red 0.64/0.33, green 0.30/0.60, blue 0.15/0.06, white
  0.3127/0.3290 — the `srgb` transfer function, luminances 0.2 / 80 / 80
  cd/m²). In the BT.2100 color mode it is `bt2020` primaries (red
  0.708/0.292, green 0.170/0.797, blue 0.131/0.046, white
  0.3127/0.3290), `st2084_pq`, and as luminances its minimum and maximum
  (from its EDID, else 0.005 and 1000 cd/m²) and the reference white,
  with the same target luminance. When a monitor's color mode changes,
  its `wp_color_management_output_v1` objects get
  `image_description_changed`. A monitor that has gone away gives a
  description that fails with `no_output`.
- Every surface's preferred description is sRGB (as above), since that
  is the composition space; `preferred_changed` is sent only if a
  surface's preferred description changes (it does not).

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

## Animations (Helix)

The compositor animates windows, shell surfaces and workspace switches
(it draws them; apps are not involved). Each animation eases out (cubic):
fast at first, slowing to the end.

| What | When | How | Length |
|------|------|-----|--------|
| Window opens | first shown, or brought back from minimized | fades in from transparent while growing from 90 % to full size about its center | 200 ms |
| Window closes | its app takes it away (`xdg_toplevel` destroyed, X11 window unmapped) | its last image fades out while shrinking to 90 % | 150 ms |
| Window minimized | | its last image shrinks to 20 % while moving to the bottom middle of its monitor (where the dock is) and fading out | 250 ms |
| Window changes place | maximized, tiled to a side, made full screen, or restored from any of these (also when the work area changes under a maximized or tiled window) | drawn in a rectangle going from where it was to where it goes; whatever the window shows meanwhile (its old or new size) is stretched to fill it | 250 ms |
| Workspace switch | | the old workspace's windows slide out a monitor's width the other way while the new one's slide in (switching to the next workspace: from the right) | 250 ms |
| Shell surface | overview, launcher, menus, dialogs, notifications and other `zwlr_layer_shell_v1` surfaces appear or go away | fade in / out | 150 ms |

- The panel, the dock (and its edge), the on-screen keyboard and the
  backgrounds appear and go away without animation.
- The last image of something going away is the textures of its surfaces
  at that moment; a window whose surfaces are already gone (for example,
  its app crashed) disappears without animation.
- Windows on every workspace do not slide. Window title bars Helix draws
  for X11 windows are not animated, except that they follow a window
  changing place (drawn at the rectangle's width).
- Changing place again part way starts from where the window is drawn
  at that moment. With `reduced-motion` a window changes place at once.
- Screenshots and screen casts show the scene as it is, not mid-animation
  states of windows going away.
- `org.gnome.desktop.interface enable-animations` false: no animations.
- `org.gnome.desktop.a11y.interface reduced-motion` `reduce`: fades only
  (nothing grows, shrinks or slides; a workspace switch crossfades).
- `HELIX_ANIMATION_SLOWDOWN` in the compositor's environment (a factor
  from 1 to 100) makes every animation that many times longer, for
  testing.

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
keys") at the pointer; `toggle-shade` rolls the window up to its title
bar, or back down; `none` does nothing.

A **shaded** window shows only its title bar: its contents are not drawn
and the pointer goes through where they were (to what lies below); its
frame's resize borders are gone. It keeps its size, place, workspace and
focus (keys still go to it). Shading is for windows Helix frames (X11
windows); it lasts until toggled again or the window closes. The X11
`_NET_WM_STATE_SHADED` state is not set.

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

The compositor itself handles:

- `org.gnome.mutter.keybindings` `switch-monitor` (default Super+P and
  `XF86Display`): the display switcher (`org.gnome.Mutter.DisplayConfig.md`,
  "Display switcher"). Not while the session is locked.

- `org.gnome.mutter.wayland.keybindings` `switch-to-session-1` … `-12`
  (default Ctrl+Alt+F1 … F12): switch to that virtual terminal (bare
  hardware only). They also work while an app inhibits shortcuts and
  while the session is locked.
- `org.gnome.mutter.keybindings` `rotate-monitor` (default
  `XF86RotateWindows`): turns the built-in panel to the next quarter turn
  (DisplayConfig transforms 0 → 1 → 2 → 3 → 0; a flipped panel stays
  flipped), applied like automatic rotation
  (`org.gnome.Mutter.DisplayConfig.md`, "Automatic rotation"), even
  while `orientation-lock` is on. Nothing happens without a built-in
  panel or while locked.
- `cancel-input-capture` (default `<Super><Shift>Escape`): ends an input
  capture (`org.freedesktop.impl.portal.md`, "InputCapture").
- Ctrl+Alt+Backspace always quits Helix; it cannot be changed.

Changes to these keys apply at once.

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

**Bells.** An app rings the bell with `xdg_system_bell_v1`; an X11 app
rings it through XKB (the compositor's own Xwayland connection selects
XKB bell notifications on the core keyboard; the notification's window
is the app's window, none for 0). With
`audible-bell` the `bell` sound of the sound theme plays (subject to
`org.gnome.desktop.sound event-sounds`). With `visual-bell`,
`visual-bell-type` `fullscreen-flash` flashes the monitor of the window
(or of the pointer) white for a moment; `frame-flash` flashes the
window itself (the focused one when the app is unknown).

**Locate pointer.** With `org.gnome.desktop.interface locate-pointer`,
pressing and releasing `locate-pointer-key` (default `Control_L`) alone
shows shrinking circles around the pointer for half a second.

## Drags and click sounds (Helix)

- The shell's own drags (a dock icon, a window in the overview) start
  once the pointer has moved `org.gnome.desktop.peripherals.mouse
  drag-threshold` pixels (default 8) with the button down.
- With `org.gnome.desktop.sound input-feedback-sounds` and `event-sounds`
  on, a click on one of the shell's dialog buttons, quick settings or
  the dock plays the sound theme's `button-pressed`. Apps make their own
  feedback sounds (GTK reads the key; X11 apps get it as XSETTINGS).
  Which of its controls GNOME's shell makes sounds for is not documented
  here.

## Run a command (Helix)

`org.gnome.desktop.wm.keybindings panel-run-dialog` (default Alt+F2)
opens a dialog, "Run a Command", with a text field "Enter a Command" and
"Cancel" / "Run" (Return runs, Escape cancels). Not with
`org.gnome.desktop.lockdown disable-command-line`.

- Up and Down in the field go back and forth through
  `org.gnome.shell command-history` (oldest first; Down past the newest
  empties the field). Tab completes the first word to the longest prefix
  the matching programs on `$PATH` share (and a space after a single
  match).
- What was typed (a leading `~/` is the home folder):
  - nothing: the dialog closes;
  - a URI (`scheme://…`) or an existing folder's absolute path: it opens
    with its default app;
  - anything else: a command line, split as a shell would split it
    (quotes, no variables or pipes) and started as an app launched from
    a command line; when its program is not on `$PATH` (or is not an
    existing file), the dialog stays with "Command not found: <program>"
    in the warning color, and other failures show their message.
- A command that ran is added to the end of `command-history` (an earlier
  copy is removed; the newest 30 are kept).

GNOME's own run dialog (its special commands such as `r` and `lg`, and
how it completes) is not documented here; this one is Helix's.

## Windows wanting attention (Helix)

A window wants attention when:

- it asks to be activated (`xdg_activation_v1`) and Helix does not grant
  it: Helix activates a window only for a token made within the last 10
  seconds (an app launched from the shell, a notification click);
- an X11 window sets the urgency flag of its `WM_HINTS` (and stops
  wanting it when it clears the flag);
- an X11 client asks to add, remove or toggle
  `_NET_WM_STATE_DEMANDS_ATTENTION` on its window (a `_NET_WM_STATE`
  client message to the root window).

A window with keyboard focus never wants attention, and one stops when it
gets focus or closes. While a window wants it:

- a notification "<window title> is ready" (the app's name and icon from
  its desktop file, else the window's app id) is shown, following the
  app's notification settings; clicking it activates the window. It is
  withdrawn when the window stops wanting attention.
- the dock marks its app as urgent (as LauncherEntry urgency does, see
  "Dock").

How GNOME words its notification and which activation requests it turns
into attention are not documented here; these are Helix's.

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
is true, pushing the pointer into the top-left corner of the primary
monitor shows the overview, or hides it when it (or the launcher) is
shown. With a mouse, touchpad, mouse keys or remote relative motion, the
pointer must be on the corner pixel and keep moving up or left past it:
the unaccelerated motion beyond the corner (not the way there) is added
up, starting over when more than a second passes between pushes or the
pointer leaves the corner pixel, and the overview toggles when it reaches
100 pixels. Pointers that cannot push past the screen edge (tablet pens,
remote absolute motion, the host pointer of a nested session) toggle it
on reaching the corner pixel. The corner works again once the pointer has
left the 32×32 pixel square at the corner. It does nothing while a pointer button
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
  `open-new-window-application-1` … `-9` (default Super+Ctrl+1 … 9)
  launch the Nth app again (a new window). While the dock is on with
  Dash to Dock's `hot-keys` (below), its keys are used instead of these
  two sets.
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
  shown and hidden on its own); otherwise on one monitor: with
  `preferred-monitor` -2 (the default) the one whose connector is
  `preferred-monitor-by-connector`, else the primary monitor (`primary`,
  the default); with -1 the primary monitor; with 0 or more that monitor
  in the order the compositor lists them (the primary monitor when there
  is no such monitor). On the edge `dock-position` names (`BOTTOM` by default;
  `TOP` is below the panel). Centered on the edge, or along all of it with
  `extend-height` (then the icons start at the edge's beginning unless
  `always-center-icons`). Otherwise it is at most `height-fraction`
  (default 0.9) of the edge's length less twice the panel's height.
- **What.** In order: the Show Apps button when `show-apps-at-top`; the
  favorites (`show-favorites`); apps with windows that are not favorites,
  in the order their first window opened (`show-running`; with
  `isolate-workspaces`, only windows of the current workspace count; with
  `isolate-monitors`, only windows whose middle is on the dock's monitor,
  which also goes for the running marks and clicks);
  mounted drives that can be unmounted or ejected (`show-mounts`:
  removable drives, discs, phones; network shares only with
  `show-mounts-network`), and without `show-mounts-only-mounted` also
  drives that are not mounted but can be (clicking one, or its menu's
  "Mount", mounts it, and it then opens); the Trash
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
  beside the bar; for an app with windows, below its name (dimmer) the
  windows' titles, as many as the dock counts for it (newest used first,
  at most five, then "N more"; "Untitled" for a window without one).
  With `hide-tooltip` nothing is shown on pointing.
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
  `autohide-in-fullscreen`. The edge is a one-pixel strip along the
  screen edge. With `require-pressure-to-show` (the default) the dock
  shows only when the pointer, on that strip, keeps pushing against the
  edge: the pointer's unaccelerated motion towards the edge
  (`zwp_relative_pointer_v1`, which is not stopped by the edge) is added
  up, starting over when more than a second passes between pushes or the
  pointer leaves the strip, and the dock shows at once when it reaches
  `pressure-threshold` pixels; without it the dock shows after
  `show-delay` seconds on the strip. Showing and hiding, the bar
  slides in from beyond its edge and back out over `animation-time`
  seconds (default 0.2, eased out; at most 2); a dock asked to show
  while sliding out turns back from where it is. It appears and goes at
  once when `org.gnome.desktop.interface enable-animations` is off, and
  when it is taken down for the overview, the launcher or the lock
  screen.
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
  `focus-minimize-or-previews` show its windows when it has several:
  with `show-windows-preview` (the default) as previews (below), else as
  a menu of their titles; `focus-minimize-or-appspread`
  minimizes when focused, else opens the overview or activates; `quit`
  closes its windows; `skip` does nothing. An app without windows is
  launched (except with `skip`). A drive opens in the file manager, the
  Trash opens, Show Apps opens or closes the launcher.
- **Scrolling** over the dock, by `scroll-action`: `cycle-windows`
  activates the next (or previous) window of the app under the pointer;
  `switch-workspace` switches to the workspace to the right (or left).
  With `scroll-switch-workspace` (the default), scrolling that
  `scroll-action` leaves unused (`do-nothing`, or `cycle-windows` away
  from an app) switches workspaces too.
- **Activating an app** (a click with `cycle-windows` and the like, or a
  hot key) on an app without focus brings up only its newest used window
  with `activate-single-window` (the default); without it, all its
  windows come up, the newest used on top.
- **Window previews** are a popup beside the app's icon (above a bottom
  dock): a tile per window, at most four in a row, each 220 pixels wide
  with the window's picture (taken when the popup opens, its outline until
  the picture comes), its title below and a close button at the top
  right. Clicking a tile activates the window; the close button closes
  it; Escape or a click outside closes the popup; the arrow keys move
  between tiles and close buttons, Return chooses.
- **Right click** opens a menu. For an app: with `show-windows-preview`
  and windows open, "All Windows" (shows the previews), else its windows
  by title (each activates it); then "Open" (no windows) or "New Window", the desktop
  file's actions (`Actions=`), the app's quicklist entries
  (`contracts/dbus/com.canonical.Unity.LauncherEntry.md`, "Quicklists";
  a submenu entry, marked with an arrow, shows its entries in place of the
  menu, under a "Back" entry; Right opens it from the keyboard, Left
  goes back),
  then "Pin to Dash" or "Unpin" (adds it to
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
  replaces its notification count. Urgent apps get an orange background.
  When an app turns urgent, the dock shows for 2 seconds
  (`show-dock-urgent-notify`, the default) and its icon wiggles along
  the bar for 1.6 seconds (`dance-urgent-applications`, the default; not
  with animations off). An app is urgent when its LauncherEntry says so
  or one of its windows wants attention (see "Windows wanting
  attention"); with `workspace-agnostic-urgent-windows` (the default),
  such windows count on every workspace, also with `isolate-workspaces`
  (the app shows, and clicking it reaches that window).
- `isolate-locations` has no effect: drives and the Trash never own file
  manager windows in Helix's dock (the file manager's windows are its
  own, whatever they show).
- **Keys.** With `hot-keys` (the default) and the dock on,
  `app-hotkey-1` … `-10` (Super+1 … Super+0) act on the first to tenth
  app of the dock (favorites, then running apps; on the first monitor's
  dock) as a click does (`click-action`), `app-shift-hotkey-N` as a
  Shift+click (`shift-click-action`) and `app-ctrl-hotkey-N` launch the
  app again (a new window). After one of them, or `shortcut` (Super+Q),
  for `shortcut-timeout` seconds (default 2) the apps' icons show their
  numbers (1 … 9, then 0) at their top left (`hotkeys-overlay`) and the
  dock is shown (`hotkeys-show-dock`; it hides again afterwards unless
  the pointer is on it). `shortcut-text` is not used.
- The remaining keys are not used.

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
- The power actions logind allows: Lock Screen, Switch User… (see
  `org.gnome.desktop.lockdown.md`, "User switching"), Suspend, Log Out…,
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
  workspace (`contracts/gsettings/workspaces.md`); `app-icon-mode` of
  `org.gnome.shell.window-switcher` shapes the window switcher
  (`switch-windows`, `switch-group`, and an application's windows reached
  from the application switcher; not the application list itself): with
  `thumbnail-only` or `both` (the default) it is a row of tiles, at most
  eight (fewer when the monitor is narrower), each a picture of the
  window (taken when the switcher opens, as for the overview; the
  window's icon at 64 pixels until it comes) with the window's title
  below, the selected tile highlighted; with `both` the window's icon (32
  pixels) sits on the picture's bottom right corner. With `app-icon-only`
  it is the list described above.
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
replaces it. Likewise (since version 6) UTF-8 text, offered as
`text/plain;charset=utf-8`, `text/plain`, `UTF8_STRING`, `STRING` and
`TEXT`. Ordinary clients have no way to capture the
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
