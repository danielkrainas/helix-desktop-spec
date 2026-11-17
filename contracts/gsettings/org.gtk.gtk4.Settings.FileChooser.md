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
  order) and writes `sort-column` and `sort-order`. `type` sorts by the
  type as the Type column shows it.
- `show-hidden`: hidden entries (names starting with `.` or ending with
  `~`) are listed; Ctrl+H flips it and writes it.
- `show-size-column`: the Size column is shown.
- `startup-mode`: with `recent`, opening files without a start folder
  (none given and none remembered) starts in Recent; with `cwd`, in the
  home folder.
- `clock-format`: modification times of today read as "14:05" (`24h`)
  or "2:05 PM" (`12h`).
- `date-format`: with `with-time`, dates other than today's are followed
  by the time ("Yesterday, 14:05", "2 Nov, 14:05").
- `show-type-column`, `type-format`: the Type column is shown, naming
  types by category (`category`, as GTK 4's chooser), description or
  MIME type.
- `window-size`, `sidebar-width`: the dialog's size and the places'
  width (logical pixels), when positive; resizing the dialog or the
  places writes them (`-1`: from the screen).

*GNOME does not document this; Helix's choice:* `expand-folders`,
`location-mode`, `view-type` and `window-position` are not followed:
Helix's chooser has no grid view, its dialog is always centered, and
it always shows the path bar (Ctrl+L or Alt+D open the location field).
