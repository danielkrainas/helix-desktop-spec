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

- `dwell-mode` `window`: a primary click. (GNOME's click type window is not
  shown; `click-type-window-visible` has no effect.)
- `dwell-mode` `gesture`: after the rest, the next move farther than the
  threshold chooses, by its main direction (`left`, `right`, `up`,
  `down`): `dwell-gesture-single` a primary click,
  `dwell-gesture-double` a double click, `dwell-gesture-secondary` a
  secondary click, `dwell-gesture-drag` a primary press (released by the
  next dwell). The pointer goes back to the resting point for it.
  Waiting `dwell-time` again without choosing gives up.
