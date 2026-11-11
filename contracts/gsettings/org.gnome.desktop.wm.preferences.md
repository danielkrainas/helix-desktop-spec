# `org.gnome.desktop.wm.preferences`

Key surface of the `org.gnome.desktop.wm.preferences` GSettings schema, described from the
installed schema via GSettings introspection (gsettings-desktop-schemas 46.1).
Verify against GNOME 51 before relying on newer keys (see `gaps.md`).

Path: `/org/gnome/desktop/wm/preferences/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `action-double-click-titlebar` | `s` | `'toggle-maximize'` | `toggle-shade`, `toggle-maximize`, `toggle-maximize-horizontally`, `toggle-maximize-vertically`, `minimize`, `none`, `lower`, `menu` |
| `action-middle-click-titlebar` | `s` | `'none'` | `toggle-shade`, `toggle-maximize`, `toggle-maximize-horizontally`, `toggle-maximize-vertically`, `minimize`, `none`, `lower`, `menu` |
| `action-right-click-titlebar` | `s` | `'menu'` | `toggle-shade`, `toggle-maximize`, `toggle-maximize-horizontally`, `toggle-maximize-vertically`, `minimize`, `none`, `lower`, `menu` |
| `audible-bell` | `b` | `true` |  |
| `auto-raise` | `b` | `false` |  |
| `auto-raise-delay` | `i` | `500` | `0` to `10000` |
| `button-layout` | `s` | `'appmenu:close'` |  |
| `disable-workarounds` | `b` | `false` |  |
| `focus-mode` | `s` | `'click'` | `click`, `sloppy`, `mouse` |
| `focus-new-windows` | `s` | `'smart'` | `smart`, `strict` |
| `mouse-button-modifier` | `s` | `'<Super>'` |  |
| `num-workspaces` | `i` | `4` | `1` to `36` |
| `raise-on-click` | `b` | `true` |  |
| `resize-with-right-button` | `b` | `false` |  |
| `theme` | `s` | `'Adwaita'` |  |
| `titlebar-font` | `s` | `'Cantarell Bold 11'` |  |
| `titlebar-uses-system-font` | `b` | `true` |  |
| `visual-bell` | `b` | `false` |  |
| `visual-bell-type` | `s` | `'fullscreen-flash'` | `fullscreen-flash`, `frame-flash` |
| `workspace-names` | `as` | `[]` |  |
