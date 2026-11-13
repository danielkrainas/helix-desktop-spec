# `org.gnome.desktop.privacy`

Key surface of the `org.gnome.desktop.privacy` GSettings schema in gsettings-desktop-schemas
51.0, described through GSettings introspection of the compiled schema.

Path: `/org/gnome/desktop/privacy/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `disable-camera` | `b` | `false` |  |
| `disable-microphone` | `b` | `false` |  |
| `disable-sound-output` | `b` | `false` |  |
| `hide-identity` | `b` | `false` |  |
| `old-files-age` | `u` | `30` |  |
| `privacy-screen` | `b` | `false` |  |
| `recent-files-max-age` | `i` | `-1` |  |
| `remember-app-usage` | `b` | `true` |  |
| `remember-recent-files` | `b` | `true` |  |
| `remove-old-temp-files` | `b` | `false` |  |
| `remove-old-trash-files` | `b` | `false` |  |
| `report-technical-problems` | `b` | `false` |  |
| `send-software-usage-stats` | `b` | `false` |  |
| `show-full-name-in-top-bar` | `b` | `true` |  |
| `usb-protection` | `b` | `true` |  |
| `usb-protection-level` | `s` | `'lockscreen'` | `lockscreen`, `always` |

## Helix

### Recent files

GTK apps read `remember-recent-files` and `recent-files-max-age`
themselves; X11 apps get them as XSETTINGS (`Gtk/RecentFilesEnabled`,
`Gtk/RecentFilesMaxAge`, see `contracts/wayland/README.md`, "X11 apps:
scaling and settings"), and Helix's file chooser follows them for its
Recent place (`org.freedesktop.impl.portal.md`, "FileChooser"). Helix does
not delete the recent files list when remembering is turned off.

### Old files

With `remove-old-trash-files` or `remove-old-temp-files` on, the shell
cleans up a minute after it starts, then every hour, and at once when
one of these keys changes after that:

- Trash: items of the home Trash (`$XDG_DATA_HOME/Trash`) trashed more
  than `old-files-age` days ago, by their `.trashinfo` `DeletionDate`
  (local time; the info file's modification time without one), are
  deleted with their info files. The Trash folders of other drives are
  left alone.
- Temporary files: in the temporary folder (`$TMPDIR`, else `/tmp`) and
  below it, the user's own regular files and symbolic links whose
  modification, access and change times are all more than
  `old-files-age` days ago are deleted, and the user's directories left
  empty that are as old. Sockets, pipes, devices and other users' files
  are left alone.
- Whatever was used within the last hour stays, even with
  `old-files-age` 0.

How gnome-settings-daemon decides what is old (which times it looks at)
and how often it cleans up are not documented here; these rules are
Helix's.

### Other keys

`remember-app-usage`, `hide-identity`, `privacy-screen`,
`report-technical-problems`, `send-software-usage-stats`,
`show-full-name-in-top-bar`, `usb-protection` and `usb-protection-level`
have no effect in Helix (it records no app usage and has no privacy screen
or USB protection support).
