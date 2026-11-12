# Interface accessibility (Helix)

How Helix's own UI (panel, menus, dialogs, launcher, overview, lock screen)
follows `org.gnome.desktop.a11y.interface`
(`org.gnome.desktop.a11y.interface.md`). Changes apply at once.

- `high-contrast`: Helix's own colors become black and white: text is
  white; shapes (backgrounds, fields, buttons, borders) are black when dark
  (luminance below 0.4) and white when light; saturated colors (the accent,
  warnings) stay; opacity is kept. Pictures (wallpapers, window pictures,
  app icons) are not changed. Apps follow the key themselves.
- `reduced-motion` `reduce`: the compositor's animations only fade
  (`contracts/wayland/README.md`, "Animations").
- `show-status-shapes`: the quick settings switches show their state
  with a shape as well as their color, before the label: a short
  vertical bar when on, a ring when off. (Check boxes already show a
  check mark; the dock's running dots are shapes.)
- `keyboard-focus-visible-timeout`: in Helix's menus (window, dock,
  input source, accessibility and IBus property menus, window previews),
  the highlight the arrow keys move goes away after that many seconds
  without another key; the entry stays selected (Return still chooses
  it) and pointing at an entry highlights it again. 0 or a negative value
  (the default, -1): the highlight stays. In window previews the arrow
  keys move between tiles and close buttons; in other menus ← and → move
  as ↑ and ↓ do.
