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

- `panel-left`, `panel-center`, `panel-right`: the panel's items in each
  zone, in order from left to right (see
  `contracts/wayland/README.md`, "Panel layout").
- `quick-settings`: the switches in the quick settings menu, in order
  (see `contracts/wayland/README.md`, "Quick settings").

Ids of the form `helix:<name>` are Helix's built-in items. Other ids are
reserved for extensions (`proposals/0001-extensions.md`), as
`<extension id>/<item id>`. Ids Helix does not know are kept in the key
but not shown, so a layout survives an extension being turned off or an
older Helix reading a newer layout.
