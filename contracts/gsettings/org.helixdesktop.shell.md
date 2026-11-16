# `org.helixdesktop.shell`

Helix's own shell settings. This is not a GNOME schema: Helix defines
it. Helix installs it as `org.helixdesktop.shell.gschema.xml` (in
`$datadir/glib-2.0/schemas/`), and this table is its contract.

Path: `/org/helixdesktop/shell/`

| Key | Type | Default |
|-----|------|---------|
| `panel-left` | `as` | `['helix:apps', 'helix:workspaces']` |
| `panel-center` | `as` | `['helix:clock']` |
| `panel-right` | `as` | `['helix:tray', 'helix:accessibility', 'helix:keyboard', 'helix:status', 'helix:power']` |
| `quick-settings` | `as` | `['helix:wifi', 'helix:bluetooth', 'helix:airplane-mode', 'helix:auto-rotate', 'helix:night-light', 'helix:dark-style', 'helix:do-not-disturb']` |
| `banner-ignore-timeout` | `u` | `10` |
| `critical-banner-ignore-timeout` | `u` | `30` |
| `app-grid-recent` | `b` | `true` |
| `allow-tearing` | `b` | `true` |

- `panel-left`, `panel-center`, `panel-right`: the panel's items in each
  zone, in order from left to right (see
  `contracts/wayland/README.md`, "Panel layout").
- `quick-settings`: the switches in the quick settings menu, in order
  (see `contracts/wayland/README.md`, "Quick settings").
- `banner-ignore-timeout`: seconds of user activity a notification banner
  without a timeout (`expire_timeout` 0) stays on screen, left alone,
  before it moves to the notification list. `critical-banner-ignore-timeout`:
  the same for critical notifications. `0` keeps such banners until the
  user acts on them. Range 0–600 (the schema's range); 1–4 count as 5.
  The shell follows changes at once. See
  `contracts/dbus/org.freedesktop.Notifications.md`, "How long banners
  stay".
- `app-grid-recent`: whether the app grid's first page starts with the
  "Recent" row of the apps launched last (see
  `contracts/wayland/README.md`, "App grid"). The row also needs
  `org.gnome.desktop.privacy remember-app-usage`; turning this key off
  only hides the row, and the recent apps stay recorded while usage is
  remembered. The shell follows changes to either key at once, also
  while the launcher is open.

- `allow-tearing`: whether a full-screen window that asks for it
  (`wp_tearing_control_v1` `async`) may be shown without waiting for
  vblank, on the DRM backend (see `contracts/wayland/README.md`,
  "Tearing"). The compositor reads it (it watches this schema too) and
  follows changes at the next frame. Off, every frame waits for vblank.

Ids of the form `helix:<name>` are Helix's built-in items. Extensions'
items are `<extension id>/<item id>` (`contracts/extensions/README.md`):
panel items in the panel keys, toggles and sliders in
`quick-settings`. Ids Helix does not know are kept in the key
but not shown, so a layout survives an extension being turned off or an
older Helix reading a newer layout.

## Older installed schemas

The installed copy of this schema (and of `org.helixdesktop.extensions`) can be
older than the programs reading it, after a partial upgrade or when a
newer build runs against an older package. Helix's programs keep
working then: a key the installed schema lacks, or gives another type,
reads as its default from the table above and is not written, and the
first such read is logged once. Losing a newer key's setting is
acceptable; stopping the shell is not.
