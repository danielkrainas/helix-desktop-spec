# Magnifier and screen reader (Helix)

How Helix applies `org.gnome.desktop.a11y.magnifier`
(`org.gnome.desktop.a11y.magnifier.md`) and the switches of
`org.gnome.desktop.a11y.applications`
(`org.gnome.desktop.a11y.applications.md`). Changes apply at once.

## Magnifier

The compositor magnifies the screen while `screen-magnifier-enabled` is
true and `mag-factor` is above 1 (color effects, below, apply while it is
on).

- Every monitor is magnified by `mag-factor` (its whole area:
  `screen-position` and `lens-mode` are not supported). The pointer is
  magnified too; input is not affected (the pointer still moves over the
  real, unmagnified layout).
- Which part of a monitor shows follows the pointer while it is on that
  monitor, per `mouse-tracking`:
  - `proportional`: the pointer's place in the shown part matches its place
    on the monitor (at the left edge the left edge shows, and so on);
  - `centered`: the pointer stays in the middle; with `scroll-at-edges`
    false, the shown part stops at the monitor's edges (the pointer then
    leaves the middle), with true it does not;
  - `push`: the shown part moves only as far as needed to keep the pointer
    in it;
  - `none`: the shown part stays where it is.
  The shown part starts as `proportional` puts it.
- `show-cross-hairs`: two lines through the pointer, `cross-hairs-thickness`
  pixels wide and `cross-hairs-length` long (both on screen), in
  `cross-hairs-color` (`#rrggbb`) at `cross-hairs-opacity`; with
  `cross-hairs-clip`, they stop short of the pointer (a gap of 16 pixels
  times `mag-factor` around it).
- Screenshots and screen casts are not magnified.
- Color effects, applied to what the magnifier shows (with each channel
  from 0 to 1), in this order:
  - `invert-lightness`: the lightness is inverted, hue and saturation
    kept: every channel `c` becomes `c + 1 - max - min` (`max`, `min` of
    the pixel's three channels);
  - `color-saturation` `s`: each channel becomes `l + (c - l) * s`, `l` the
    luminance `0.2126 r + 0.7152 g + 0.0722 b`;
  - `contrast-red`/`-green`/`-blue` `k` and `brightness-red`/`-green`/`-blue`
    `b`: `(c - 0.5) * (1 + k) + 0.5 + b`, kept within 0 and 1.
  They apply while the magnifier is on, also at `mag-factor` 1.
- Not supported: `focus-tracking` and `caret-tracking` (they need the
  accessibility bus).

## Keys

Keys of `org.gnome.settings-daemon.plugins.media-keys` (handled by the
shell, also while locked):

| Key | Action |
|-----|--------|
| `magnifier` | flip `screen-magnifier-enabled` |
| `magnifier-zoom-in` | turn the magnifier on, or raise `mag-factor` by 0.5 (up to 32) when it is on |
| `magnifier-zoom-out` | lower `mag-factor` by 0.5; at 1 or below, `mag-factor` becomes 1 and the magnifier turns off |
| `screenreader` | flip `screen-reader-enabled` |
| `on-screen-keyboard` | flip `screen-keyboard-enabled` |

## Screen reader and on-screen keyboard

Helix starts no screen reader or on-screen keyboard itself. Apps that
provide them install autostart files whose `AutostartCondition` names the
switch (Orca: `GSettings org.gnome.desktop.a11y.applications
screen-reader-enabled`); the session starts and stops them as the switch
changes (`org.gnome.SessionManager.md`, "Autostart").
