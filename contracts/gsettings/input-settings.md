# Input settings — how Helix applies them

Behavior for `org.gnome.desktop.input-sources`,
`org.gnome.desktop.peripherals.{keyboard,mouse,touchpad}` and the
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
- `switch-input-source` / `switch-input-source-backward` open the input
  source switcher. It lists the sources most recently used first (the
  `mru-sources` entries that are in `sources`, then the other sources in
  `sources` order) by id, with the second one selected (the last one,
  backward). Pressing the key again moves the selection (backward with
  the `-backward` key or Shift). Releasing the modifier activates the
  selected source by moving it to the front of `mru-sources`; Escape
  cancels. So a quick press and release goes back to the previous source.
  A binding without a modifier switches at once. With a single source
  there is nothing to switch to.

## Key repeat

`repeat` turns it on or off; `delay` is the time in milliseconds before a
held key repeats, `repeat-interval` the time between repeats.

## Mouse and touchpad

On bare hardware (libinput devices). A pointer device that supports
tapping is a touchpad; other pointer devices use the mouse settings.
Nested sessions leave pointer settings to the host.

- Both: `speed`, `accel-profile` (`default` and `custom` mean the device's
  default profile), `natural-scroll`, `middle-click-emulation`,
  `left-handed` (for the touchpad, `mouse` follows the mouse's setting).
- Touchpad also: `tap-to-click`, `tap-and-drag`, `tap-and-drag-lock`,
  `tap-button-map`, `two-finger-scrolling-enabled` and
  `edge-scrolling-enabled` (two-finger wins when both are on),
  `disable-while-typing`, `click-method`, `send-events`.
