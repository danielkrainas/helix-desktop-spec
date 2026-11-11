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

## Key repeat

`repeat` turns it on or off; `delay` is the time in milliseconds before a
held key repeats, `repeat-interval` the time between repeats.

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
## Touchscreens

Touch input goes to the surface under the touch point, with Wayland touch
events (`wl_touch`); touching a window also focuses and raises it, as a
click does. Shell surfaces (panel, launcher, dialogs) take a tap as a
click. A touchscreen covers one monitor: the built-in panel if there is
one, else the primary monitor. The per-device
`org.gnome.desktop.peripherals.touchscreen` settings (`output`) are not
read (see `gaps.md`); there are no gestures.
