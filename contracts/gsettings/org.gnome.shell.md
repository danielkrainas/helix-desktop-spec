# `org.gnome.shell`

Key surface of the `org.gnome.shell` GSettings schema in gnome-shell
51.0, described through GSettings introspection of the compiled schema.

Path: `/org/gnome/shell/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `allow-extension-installation` | `b` | `true` |  |
| `always-show-log-out` | `b` | `false` |  |
| `app-picker-layout` | `aa{sv}` | (GNOME's default app grid, generated at build time) |  |
| `command-history` | `as` | `[]` |  |
| `development-tools` | `b` | `true` |  |
| `disable-extension-version-validation` | `b` | `false` |  |
| `disable-user-extensions` | `b` | `false` |  |
| `disabled-extensions` | `as` | `[]` |  |
| `enabled-extensions` | `as` | `[]` |  |
| `favorite-apps` | `as` | `['org.gnome.Epiphany.desktop', 'org.gnome.Calendar.desktop', 'org.gnome.Nautilus.desktop', 'org.gnome.Software.desktop', 'org.gnome.TextEditor.desktop', 'org.gnome.Calculator.desktop']` |  |
| `last-selected-power-profile` | `s` | `'power-saver'` |  |
| `looking-glass-history` | `as` | `[]` |  |
| `remember-mount-password` | `b` | `false` |  |
| `welcome-dialog-last-shown-version` | `s` | `''` |  |

## Helix

Keys not covered elsewhere:

- `remember-mount-password`: the first state of the "Remember Password"
  box in the shell's mount password dialogs
  (`contracts/gsettings/removable-media.md`, "Questions"); the box's state
  when such a dialog is answered is written back, so it stays as the user
  left it.
- `last-selected-power-profile`: when the user picks a power profile other
  than `balanced` in quick settings, it is written here. Helix's quick
  settings show every profile, so it reads nothing from it.
- `always-show-log-out`: no effect; Helix's Power menu always has
  "Log Out…" (unless `org.gnome.desktop.lockdown disable-log-out`).
- `command-history`: see `contracts/wayland/README.md`, "Run a command".

### Welcome dialog

A few seconds after the shell starts, when `welcome-dialog-last-shown-version`
is older than the shell's version (`51.0`; versions compare by their
dot-separated numbers, the empty string being the oldest, so a huge number
turns this off) and GNOME Tour (`org.gnome.Tour.desktop`) is installed,
a dialog asks "Would you like a short tour of the desktop?" (title
"Welcome") with "No Thanks" and "Take Tour", which starts GNOME Tour.
Either way the key is then set to the shell's version. GNOME's own
welcome dialog's wording and look are not documented here.
