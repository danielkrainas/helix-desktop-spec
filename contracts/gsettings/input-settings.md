# Input settings — how Helix applies them

Behavior for `org.gnome.desktop.input-sources`,
`org.gnome.desktop.peripherals.{keyboard,mouse,touchpad,trackball,pointingstick}` and the
input-source keys of `org.gnome.desktop.wm.keybindings` (key tables in the
files named after each schema). Changes apply at once.

## Keyboard layouts

- `sources`: the `xkb` entries make up the keymap, in order. An entry's id
  is `layout` or `layout+variant`. `xkb-model` and `xkb-options` apply to
  the whole keymap. With no `xkb` entries the compositor keeps its default
  keymap.
- The active source is the first entry of `mru-sources` that is also in
  `sources`; if there is none, the first source. `current` is deprecated
  and ignored.
- `per-window`: when true, each window keeps its own active source. A
  window that gets focus for the first time keeps the source active then;
  switching sources while it has focus changes its source; when a window
  with a remembered source gets focus again, that source becomes active
  again (moved to the front of `mru-sources`).

## Input methods (`ibus` sources)

An `ibus` entry's id is an IBus engine name (e.g. `anthy`, `hangul`,
`table:cangjie5`). While one is the active source:

- The shell runs IBus: it starts `ibus-daemon` (with its panel disabled)
  on a private address when the session has `ibus` sources and none is
  running yet, and makes the engine IBus's global engine
  (`contracts/dbus/org.freedesktop.IBus.md`).
- Apps type through the Wayland text-input protocol, as under GNOME: no
  environment variables are needed. When a text field has focus, the
  shell (the session's input method) passes the keys to IBus, gives
  committed and pre-edit text to the app, and shows IBus's candidates in
  a popup next to the text cursor. Keys IBus does not use go to the app
  unchanged.
- Engine properties (the input context's `RegisterProperties` and
  `UpdateProperty`; the shell asks for them with capability 16): while an
  `ibus` source is active, the input source menu lists them below the
  sources, after a separator. Toggles are check boxes (checked at state
  1), radio items show a dot when chosen, a menu is a submenu (marked
  with an arrow; it opens in place of the menu under a "Back" entry, as
  the dock's quicklist submenus do; menus in menus too; a menu with no
  entries left is left out), separators separate; hidden and insensitive
  properties are left out. Choosing one calls `PropertyActivate` (a
  toggle with the other state, a radio item with 1, others with 0). When
  a property named `InputMode` has a symbol (as `あ` or `A`), the panel's
  indicator shows that symbol, also with a single source.
- Emoji and Unicode entry (IBus's own shortcuts, such as Ctrl+Shift+E
  and Ctrl+Shift+U) are IBus's: its GTK extension shows their windows
  as an app; the shell takes no part.
- The keymap is the engine's own layout, as IBus describes the engine:
  its `<layout>`, `<layout_variant>` and `<layout_option>` in an IBus
  component file (`ibus/component/*.xml` under the system data
  directories, `XDG_DATA_DIRS`), with `xkb-model` and `xkb-options`
  (the engine's options added). When the engine's layout is `default`, or
  no component file lists the engine (some components list their engines
  by running a program, which Helix does not do), the keymap stays that
  of the most recently used `xkb` source.

While an `xkb` source is active, keys go straight to the app.
- `switch-input-source` / `switch-input-source-backward` open the input
  source switcher. It lists the sources most recently used first (the
  `mru-sources` entries that are in `sources`, then the other sources in
  `sources` order) by name (the layout's description from the system's
  XKB rules, `evdev.xml`, e.g. "German (no dead keys)"; the id when it is
  not listed), with the second one selected (the last one,
  backward). Pressing the key again moves the selection (backward with
  the `-backward` key or Shift). Releasing the modifier activates the
  selected source by moving it to the front of `mru-sources`; Escape
  cancels. So a quick press and release goes back to the previous source.
  A binding without a modifier switches at once. With a single source
  there is nothing to switch to.

## Input source indicator (Helix)

With two or more input sources, the panel shows the active one's short
name (`helix:keyboard`, see `contracts/wayland/README.md`, "Panel
layout"; by default left of the system status icons):

- an `xkb` source: its `shortDescription` in the XKB rules
  (`rules/evdev.xml`), the variant's, else its layout's (e.g. `en`, `de`);
  without one, the layout id;
- an `ibus` source: the engine name up to its first `:` or `-`, at most 3
  letters (e.g. `anthy` → `ant`, `xkb:us::eng` → `xkb`).

Clicking it opens a menu of the sources (most recently used first, each by
its description, the active one highlighted); choosing one activates it as
the switch keys do. Escape or a click outside closes the menu. The
indicator follows source changes made elsewhere within a second.

## Key repeat

`repeat` turns it on or off; `delay` is the time in milliseconds before a
held key repeats, `repeat-interval` the time between repeats.

## Num Lock and lock key lights

With `remember-numlock-state` on, Num Lock is set to `numlock-state` when
the session starts, and each change of Num Lock (by key) is written back
to `numlock-state`; with it off, neither happens. Num Lock and Caps Lock
stay as they were when the keyboard layout changes. Keyboards' Num Lock,
Caps Lock and Scroll Lock lights follow the lock states (bare hardware;
a keyboard that appears gets the current state). The lights have not
been tried on real keyboards (see `gaps.md`).

## Pointing devices

On bare hardware (libinput devices). A pointer device that supports
tapping is a touchpad; one udev tags `ID_INPUT_TRACKBALL` is a trackball,
`ID_INPUT_POINTINGSTICK` a pointing stick; other pointer devices are
mice. Nested sessions leave pointer settings to the host.

- Mice: `speed`, `accel-profile`, `custom-accel-config`, `natural-scroll`,
  `middle-click-emulation`, `left-handed`,
  `scroll-wheel-emulation-button` and `-lock`.
- Touchpads: the same pointer keys of the touchpad schema (`left-handed`
  `mouse` follows the mouse's setting), and `tap-to-click`,
  `tap-and-drag`, `tap-and-drag-lock`, `tap-button-map`,
  `two-finger-scrolling-enabled` and `edge-scrolling-enabled` (two-finger
  wins when both are on), `disable-while-typing`, `click-method`,
  `send-events`, `scroll-speed`.
- Trackballs: `accel-profile`, `custom-accel-config`,
  `middle-click-emulation`, `scroll-wheel-emulation-button` and `-lock`
  from the trackball schema; `speed`, `natural-scroll` and `left-handed`
  from the mouse schema.
- Pointing sticks: `speed`, `accel-profile`, `custom-accel-config` and
  `scroll-method` (`on-button-down`: scroll while the middle button is
  held; `none`; `default`: the device's default) from the pointing stick
  schema; `natural-scroll` and `left-handed` from the mouse schema.

Details:

- `accel-profile` `default` means the device's default profile; `custom`
  uses `custom-accel-config`: `pointer-step` (`d`) and `pointer-speeds`
  (`ad`) are the points of libinput's custom acceleration function for
  pointer motion (`scroll-step` / `scroll-speeds`, when present, for
  scrolling). Devices without custom profiles (libinput older than 1.23)
  keep their default.
- `scroll-wheel-emulation-button` (0 off) is an X button number: 1 left,
  2 middle, 3 right, 8 back, 9 forward, 10 and up the following buttons.
  While it is held, moving the pointer scrolls; with `-lock`, a click
  toggles that instead.
- `scroll-speed` multiplies the touchpad's finger scrolling (1 unchanged).
## Touchscreens and tablets

Key tables: [`org.gnome.desktop.peripherals.touchscreen`](org.gnome.desktop.peripherals.touchscreen.md),
[`org.gnome.desktop.peripherals.tablet`](org.gnome.desktop.peripherals.tablet.md),
[`org.gnome.desktop.peripherals.tablet.stylus`](org.gnome.desktop.peripherals.tablet.stylus.md).
These schemas are relocatable, one copy per device. Where GNOME keeps the
copies is not documented (see `gaps.md`); Helix reads them at:

- touchscreens: `/org/gnome/desktop/peripherals/touchscreens/<vid>:<pid>/`
- tablets: `/org/gnome/desktop/peripherals/tablets/<vid>:<pid>/`
- styli: `/org/gnome/desktop/peripherals/stylus/<serial>/`, or for a stylus
  without a serial number `/org/gnome/desktop/peripherals/stylus/default-<vid>:<pid>/`

`<vid>` and `<pid>` are the device's USB vendor and product ids as four
lower-case hex digits; `<serial>` is the stylus's serial in lower-case hex
without leading zeros. Devices without USB ids use the defaults.

A monitor named by an `output` setting is `[vendor, product, serial]` as
DisplayConfig reports them (`org.gnome.Mutter.DisplayConfig.md`, "Monitor
identity"); all three empty means "choose automatically".

### Touchscreens

Touch input goes to the surface under the touch point, with Wayland touch
events (`wl_touch`); touching a window also focuses and raises it, as a
click does. Shell surfaces (panel, launcher, dialogs) take a tap as a
click. A touchscreen covers one monitor: the one its `output` names (read
again at each touch down), else the built-in panel if there is one, else
the primary monitor. `orientation-lock` is described in
`org.gnome.Mutter.DisplayConfig.md` ("Automatic rotation"). Swipes with
three or more fingers and swipes in from an edge are Helix's own
(`contracts/wayland/README.md`, "Gestures").

### Tablets

Settings are read when a tool comes near the tablet (pad buttons: at
each press). Pad button settings live below the tablet's, one per
button: `/org/gnome/desktop/peripherals/tablets/<vid>:<pid>/button<N>/`,
where `<N>` is `A` for the pad's first button, `B` for the second and so
on (past `Z`, the button's number counting from 1).

#### Where a tool's input goes

The compositor offers the tablet protocol, `zwp_tablet_manager_v2`
version 1 (tablets, tools and pads; no dials). For each tool in
proximity:

- **Position.** `mapping` `absolute` (the default): the tablet covers
  one monitor, the one `output` names, else the primary monitor; the
  tool's position on the tablet is a position on that monitor:
  - `left-handed`: the tablet is turned around (both axes reversed);
  - `area`: margins cut off the left, right, top and bottom of the
    tablet (fractions of its width or height; values above 1 are taken
    as percentages; each at most 0.45); outside the remaining area the
    tool does nothing;
  - `keep-aspect`: the area is cut further, from the right or the
    bottom, to the monitor's aspect ratio (using the tablet's physical
    size from libinput; without one, nothing is cut).

  `mapping` `relative`: the tool moves the position like a mouse, 4
  logical pixels per millimetre of tool motion.
- **Apps that take tablet events** (the surface under the tool belongs
  to a client that created a `zwp_tablet_seat_v2`), outside pointer
  grabs and X11 window frames: the client gets the tool's
  `proximity_in`/`proximity_out` as the tool moves onto and off its
  surfaces, `motion`, `pressure`, `distance`, `tilt`, `rotation`,
  `slider` and `wheel` as they change, `down`/`up` for the tip, `button`
  for stylus buttons, and a `frame` after each. The pointer follows the
  tool but the client is not sent pointer events while the tool is over
  it; the cursor is the one the client sets with
  `zwp_tablet_tool_v2.set_cursor`, or the named cursor it asks for with
  `wp_cursor_shape_manager_v1.get_tablet_tool_v2` (taken only while the
  tool is over one of that app's surfaces; the default cursor until it
  sets one).
  The tip touching a window focuses (and with `raise-on-click`, raises)
  it, as a click does.
- **Elsewhere** the tool drives the pointer: its tip (and the eraser's)
  is the left button.
- **Pressure** goes through the stylus's `pressure-range` (for the
  eraser end: `eraser-pressure-range`) and `pressure-curve`
  (`eraser-pressure-curve`): the range's percentages of the physical
  pressure become the whole logical range (below it 0, above it full);
  the curve is a cubic Bézier curve from (0, 0) to (100, 100) through
  the two control points the four values give, read as output pressure
  for input pressure. Malformed values count as a straight line over the
  whole range.
- **Stylus buttons** (libinput's first, second and third) follow
  `button-action`, `secondary-button-action` and
  `tertiary-button-action`: `middle`, `right`, `back`, `forward` give that
  button (to apps that take tablet events, as the tool's button with that
  code); `default` is middle, right and back for the three buttons;
  `switch-monitor` maps the tablet to the next monitor (left to right,
  wrapping; until the session ends); `keybinding` types the button's
  `button-keybinding` (`secondary-button-keybinding`,
  `tertiary-button-keybinding`) on press, as a pad button's
  `keybinding` does (below); with an empty one the button does nothing.
- **Eraser button.** On pens whose eraser is a firmware button (rather
  than the pen's other end), `eraser-button-mode` `button` asks libinput
  to report that button as the stylus's third button (`BTN_STYLUS3`)
  instead of turning the pen into an eraser; that button then follows
  `eraser-button-action` and `eraser-button-keybinding` (as above;
  `default` is back) instead of `tertiary-button-action`. `default`
  leaves libinput's behavior. This needs libinput 1.29 or later: Helix
  looks the configuration functions up when a tool first comes near
  (only with the DRM backend) and, without them, logs once and ignores
  the setting. Settings are applied again when a tool comes near after
  they changed.
- Tools are announced (`tool_added`) when first seen in proximity;
  tablets and pads when plugged in, with their name, USB ids and device
  node (`/dev/input/event*`).

#### Pads

A pad's events go to the app with the keyboard focus, when it created a
tablet seat (`enter`/`leave` follow the keyboard focus; after `enter`,
each mode group's current mode). Each mode group lists its buttons,
rings and strips as libinput groups them, and its number of modes;
`mode_switch` is sent when libinput reports a group's mode changed.

- **Buttons** follow their `action`:
  - `none` (the default): the button goes to the app
    (`zwp_tablet_pad_v2.button`);
  - `help`: an on-screen display (the shell's `ShowOSD`, icon
    `input-tablet-symbolic`) lists every button as "Button A: …" with
    "Show this help", "Switch monitor", "Send keys <keybinding>", or for
    buttons left to the app what the app says the button does
    (`zwp_tablet_pad_v2.set_feedback`), else "Application defined"; then
    the rings and strips the app described (`set_feedback` on them), as
    "Ring 1: …" and "Strip 1: …". Descriptions count only from the app
    the pad's focus is on, are kept until the focus moves to another app,
    and an empty one withdraws the control's; the `serial` is not
    checked;
  - `switch-monitor`: maps the pad's tablet to the next monitor, as the
    stylus action does;
  - `keybinding`: on press, types `keybinding` (an accelerator such as
    `<Control>z`) to the focused app: the key that gives its key name in
    the current layout, with Control, Alt, Shift and Super held as it
    says (Shift also when the key needs it). An empty `keybinding`
    counts as `none`.
- **Rings** go to the app: `angle` in degrees, `stop` when the finger
  lets go, `source` `finger` when libinput says so, then `frame`.
- **Strips** go to the app: `position` from 0 to 65535, `stop`,
  `source`, `frame`.

#### Not covered

- **Not yet tested with a real tablet.** The protocol is checked with a
  Wayland client in Helix's tests, and the pressure curve and pad
  settings by unit tests; no tablet hardware was available
  (`gaps.md`).
