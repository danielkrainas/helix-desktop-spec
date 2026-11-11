# `org.gnome.desktop.interface`

Key surface of the `org.gnome.desktop.interface` GSettings schema, described from the
installed schema via GSettings introspection (gsettings-desktop-schemas 46.1).
Verify against GNOME 51 before relying on newer keys (see `gaps.md`).

Path: `/org/gnome/desktop/interface/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `avatar-directories` | `as` | `[]` |  |
| `can-change-accels` | `b` | `false` |  |
| `clock-format` | `s` | `'24h'` | `24h`, `12h` |
| `clock-show-date` | `b` | `true` |  |
| `clock-show-seconds` | `b` | `false` |  |
| `clock-show-weekday` | `b` | `false` |  |
| `color-scheme` | `s` | `'default'` | `default`, `prefer-dark`, `prefer-light` |
| `cursor-blink` | `b` | `true` |  |
| `cursor-blink-time` | `i` | `1200` | `100` to `2500` |
| `cursor-blink-timeout` | `i` | `10` | `1` to `2147483647` |
| `cursor-size` | `i` | `24` |  |
| `cursor-theme` | `s` | `'Adwaita'` |  |
| `document-font-name` | `s` | `'Cantarell 11'` |  |
| `enable-animations` | `b` | `true` |  |
| `enable-hot-corners` | `b` | `true` |  |
| `font-antialiasing` | `s` | `'grayscale'` | `none`, `grayscale`, `rgba` |
| `font-hinting` | `s` | `'slight'` | `none`, `slight`, `medium`, `full` |
| `font-name` | `s` | `'Noto Sans 9'` |  |
| `font-rgba-order` | `s` | `'rgb'` | `rgba`, `rgb`, `bgr`, `vrgb`, `vbgr` |
| `gtk-color-palette` | `s` | `'black:white:gray50:red:purple:blue:light blue:green:yellow:orange:lavender:brown:goldenrod4:dodger blue:pink:light green:gray10:gray30:gray75:gray90'` |  |
| `gtk-color-scheme` | `s` | `''` |  |
| `gtk-enable-primary-paste` | `b` | `true` |  |
| `gtk-im-module` | `s` | `''` |  |
| `gtk-im-preedit-style` | `s` | `'callback'` |  |
| `gtk-im-status-style` | `s` | `'callback'` |  |
| `gtk-key-theme` | `s` | `'Default'` |  |
| `gtk-theme` | `s` | `'Greybird'` |  |
| `gtk-timeout-initial` | `i` | `200` |  |
| `gtk-timeout-repeat` | `i` | `20` |  |
| `icon-theme` | `s` | `'elementary-xfce-dark'` |  |
| `locate-pointer` | `b` | `false` |  |
| `menubar-accel` | `s` | `'F10'` |  |
| `menubar-detachable` | `b` | `false` |  |
| `menus-have-tearoff` | `b` | `false` |  |
| `monospace-font-name` | `s` | `'Monospace 10'` |  |
| `overlay-scrolling` | `b` | `true` |  |
| `scaling-factor` | `u` | `0` |  |
| `show-battery-percentage` | `b` | `false` |  |
| `text-scaling-factor` | `d` | `1.0` | `0.5` to `3.0` |
| `toolbar-detachable` | `b` | `false` |  |
| `toolbar-icons-size` | `s` | `'large'` | `small`, `large` |
| `toolbar-style` | `s` | `'both-horiz'` | `both`, `both-horiz`, `icons`, `text` |
| `toolkit-accessibility` | `b` | `false` |  |
