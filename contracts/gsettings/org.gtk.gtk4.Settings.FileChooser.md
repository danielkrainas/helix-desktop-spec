# `org.gtk.gtk4.Settings.FileChooser`

Key surface of GTK 4's file chooser settings schema, as GTK 4.22
installs it (Ubuntu package `libgtk-4-common` 4.22.4), described through
GSettings introspection of the compiled schema. It is not a
gsettings-desktop-schemas schema: GTK 4 apps keep their own file
chooser's state in it, and Helix's file chooser (the shell's dialog
behind the FileChooser portal, `contracts/dbus/org.freedesktop.impl.portal.md`,
"FileChooser") reads and writes the keys below so both choosers share
the user's choices.

Path: `/org/gtk/gtk4/settings/file-chooser/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `clock-format` | `s` | `'24h'` | `24h`, `12h` |
| `date-format` | `s` | `'regular'` | `regular`, `with-time` |
| `expand-folders` | `b` | `false` |  |
| `location-mode` | `s` | `'path-bar'` | `path-bar`, `filename-entry` |
| `show-hidden` | `b` | `false` |  |
| `show-size-column` | `b` | `true` |  |
| `show-type-column` | `b` | `true` |  |
| `sidebar-width` | `i` | `-1` |  |
| `sort-column` | `s` | `'name'` | `name`, `size`, `type`, `modified` |
| `sort-directories-first` | `b` | `true` |  |
| `sort-order` | `s` | `'ascending'` | `ascending`, `descending` |
| `startup-mode` | `s` | `'recent'` | `recent`, `cwd` |
| `type-format` | `s` | `'category'` | `mime`, `description`, `category` |
| `view-type` | `s` | `'list'` | `list`, `grid` |
| `window-position` | `(ii)` | `(-1, -1)` |  |
| `window-size` | `(ii)` | `(-1, -1)` |  |

## Helix

The file chooser reads these keys each time it opens; without the schema
(GTK 4 not installed) it uses the defaults above.

- `sort-column`, `sort-order`, `sort-directories-first`: how folders are
  sorted. Clicking a column header changes the column (or flips the
  order) and writes `sort-column` and `sort-order`. `type` is read as
  `name` (Helix has no Type column).
- `show-hidden`: hidden entries (names starting with `.` or ending with
  `~`) are listed; Ctrl+H flips it and writes it.
- `show-size-column`: the Size column is shown.
- `startup-mode`: with `recent`, opening files without a start folder
  (none given and none remembered) starts in Recent; with `cwd`, in the
  home folder.
- `clock-format`: modification times of today read as "14:05" (`24h`)
  or "2:05 PM" (`12h`).

*GNOME does not document this; Helix's choice:* `date-format`,
`expand-folders`, `location-mode`, `show-type-column`, `sidebar-width`,
`type-format`, `view-type`, `window-position` and `window-size` are not
followed: Helix's chooser has no Type column, no grid view and no
resizable window (its size comes from the screen).
