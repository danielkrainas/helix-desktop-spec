# Mouse accessibility (Helix)

How Helix applies `org.gnome.desktop.a11y.mouse`
(`org.gnome.desktop.a11y.mouse.md`). The compositor applies it to pointer
input from devices (not to input injected by remote desktop sessions).
Changes apply at once. Distances are logical pixels; `dwell-threshold`
is the movement allowed while "holding still" for both features.

## Simulated secondary click (`secondary-click-enabled`)

- A primary-button press is held back while the pointer stays within
  `dwell-threshold` of where it was pressed.
- Released before `secondary-click-time` seconds: a normal primary click
  (press and release).
- Held that long: on release, a secondary click (press and release of the
  secondary button) instead.
- Moving farther first: the primary press goes through at once (dragging
  works as usual).

## Dwell click (`dwell-click-enabled`)

A click happens when the pointer rests: once it has moved, staying within
`dwell-threshold` for `dwell-time` seconds triggers it at the resting
point. The next one needs the pointer to move farther than the threshold
first. While a button is held down, dwelling does nothing.

- `dwell-mode` `window`: the click type chosen in the click type window
  (below): `single` a primary click (the default), `double` a double
  click, `secondary` a secondary click, `drag` a primary press (released
  by the next dwell). The type is used once, then it is `single` again.
  Dwells on the click type window itself are always single clicks and
  keep the chosen type.
- `dwell-mode` `gesture`: after the rest, the next move farther than the
  threshold chooses, by its main direction (`left`, `right`, `up`,
  `down`): `dwell-gesture-single` a primary click,
  `dwell-gesture-double` a double click, `dwell-gesture-secondary` a
  secondary click, `dwell-gesture-drag` a primary press (released by the
  next dwell). The pointer goes back to the resting point for it.
  Waiting `dwell-time` again without choosing gives up.

## Click type window

While `dwell-click-enabled` is on, `dwell-mode` is `window` and
`click-type-window-visible` is on, the shell shows a column of four
buttons at the middle of the right edge of the primary monitor, above
windows (layer namespace `helix-click-type`): **Single**, **Double**,
**Drag**, **Secondary**, the current type highlighted. Choosing one (by
a click or a dwell) makes it the type of the next dwell click.

The shell and the compositor share the type through
`org.helix.ClickType` (Helix-private), served by the compositor on its
`org.gnome.Mutter.DisplayConfig` bus name at `/org/helix/ClickType`:

- `SetClickType(s type)`: `single`, `double`, `drag` or `secondary`
  (anything else fails with `org.freedesktop.DBus.Error.InvalidArgs`);
- property `ClickType` (`s`), with `PropertiesChanged` when it changes
  (chosen, or back to `single` after a dwell used it).

