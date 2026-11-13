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

### Privacy screen

With `privacy-screen` true, every monitor connector that has the kernel's
`privacy-screen sw-state` property is set to `Enabled` (else `Disabled`),
when the key changes and when a monitor appears (bare hardware only). A
hardware-locked privacy screen (its `privacy-screen hw-state`) may not
follow. How mutter reports privacy screens to GNOME Settings is not
documented here; Helix reports nothing about them in DisplayConfig. Not
tried on a panel with a privacy screen.

### USB protection

With `usb-protection` on, the shell follows USBGuard (bus name
`org.usbguard1` on the system bus; `org.usbguard.Devices1` at
`/org/usbguard1/Devices`): for each `DevicePresenceChanged` with event 1
(inserted), while the screen is locked (`usb-protection-level`
`lockscreen`) or always (`always`), it calls `applyDevicePolicy(id, 2,
false)` (reject, not remembered) and shows a notification "USB device
blocked". Devices whose rule names a HID boot keyboard interface
(`03:01:01`) are left alone, so a keyboard can still unlock the screen;
devices already present are left alone. Without USBGuard nothing
happens. Only on bare hardware. How gnome-settings-daemon applies these
keys (USBGuard parameters it changes, its notifications) is not
documented here; these rules are Helix's. Tested against a fake
USBGuard only.

### App usage

With `remember-app-usage` (the default), each launch from the shell (the
launcher, the app grid, the dash and the dock) is counted in
`$XDG_DATA_HOME/helix/app-usage` (lines `<desktop id>\t<count>\t<last
launch, Unix seconds>`). The launcher's search orders apps that match
equally well by use: their launches, each weighing half as much for
every 30 days since the last one. Turned off, nothing is counted, and the
file is deleted. GNOME's own record (and its effect on its search and
app grid) is not read or written.

### Other keys

`hide-identity`, `report-technical-problems`, `send-software-usage-stats`
and `show-full-name-in-top-bar` have no effect in Helix.
