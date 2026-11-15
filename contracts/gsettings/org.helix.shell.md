# `org.helix.shell`

Helix's own shell settings. This is not a GNOME schema: Helix defines
it. Helix installs it as `org.helix.shell.gschema.xml` (in
`$datadir/glib-2.0/schemas/`), and this table is its contract.

Path: `/org/helix/shell/`

| Key | Type | Default |
|-----|------|---------|
| `panel-left` | `as` | `['helix:apps', 'helix:workspaces']` |
| `panel-center` | `as` | `['helix:clock']` |
| `panel-right` | `as` | `['helix:tray', 'helix:accessibility', 'helix:keyboard', 'helix:status', 'helix:power']` |
| `quick-settings` | `as` | `['helix:wifi', 'helix:bluetooth', 'helix:airplane-mode', 'helix:auto-rotate', 'helix:night-light', 'helix:dark-style', 'helix:do-not-disturb']` |
| `banner-ignore-timeout` | `u` | `10` |
| `critical-banner-ignore-timeout` | `u` | `30` |

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

Ids of the form `helix:<name>` are Helix's built-in items. Extensions'
items are `<extension id>/<item id>` (`contracts/extensions/README.md`):
panel items in the panel keys, toggles and sliders in
`quick-settings`. Ids Helix does not know are kept in the key
but not shown, so a layout survives an extension being turned off or an
older Helix reading a newer layout.
