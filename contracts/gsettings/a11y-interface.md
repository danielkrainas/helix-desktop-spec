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
- `show-status-shapes` and `keyboard-focus-visible-timeout` have no effect.
