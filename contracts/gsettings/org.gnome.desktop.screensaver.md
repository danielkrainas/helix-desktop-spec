# `org.gnome.desktop.screensaver`

Key surface of the `org.gnome.desktop.screensaver` GSettings schema in gsettings-desktop-schemas
51.0, described through GSettings introspection of the compiled schema.

Path: `/org/gnome/desktop/screensaver/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `color-shading-type` | `s` | `'solid'` | `solid`, `vertical`, `horizontal` |
| `embedded-keyboard-command` | `s` | `''` |  |
| `embedded-keyboard-enabled` | `b` | `false` |  |
| `idle-activation-enabled` | `b` | `true` |  |
| `lock-delay` | `u` | `0` |  |
| `lock-enabled` | `b` | `true` |  |
| `logout-command` | `s` | `''` |  |
| `logout-delay` | `u` | `7200` |  |
| `logout-enabled` | `b` | `false` |  |
| `picture-opacity` | `i` | `100` | `0` to `100` |
| `picture-options` | `s` | `'zoom'` | `none`, `wallpaper`, `centered`, `scaled`, `stretched`, `zoom`, `spanned` |
| `picture-uri` | `s` | `'file://@datadir@/backgrounds/gnome/adwaita-timed.xml'` |  |
| `primary-color` | `s` | `'#023c88'` |  |
| `restart-enabled` | `b` | `false` |  |
| `secondary-color` | `s` | `'#5789ca'` |  |
| `show-full-name-in-top-bar` | `b` | `true` |  |
| `status-message-enabled` | `b` | `true` |  |
| `user-switch-enabled` | `b` | `true` |  |

## Helix

Besides `lock-enabled`, `lock-delay` and the picture keys (see
`contracts/dbus/org.gnome.ScreenSaver.md` and `background.md`):

- `user-switch-enabled`: with it on (and "Switch User…" available, see
  `org.gnome.desktop.lockdown.md`, "User switching"), the lock screen has a
  "Switch User…" button at its bottom right, which asks the display
  manager for its login screen (the session stays locked). Read each time
  the lock screen is drawn.
- `logout-enabled`, `logout-delay`, `logout-command`,
  `status-message-enabled`, `idle-activation-enabled` and the
  `embedded-keyboard-*` keys are marked deprecated and ignored in the
  GNOME 51 schema; Helix ignores them too.
- `show-full-name-in-top-bar` concerns GNOME's screen shield, shown
  before its unlock dialog; Helix's lock screen is the unlock dialog,
  which always shows the user name (as GNOME's does).
- `restart-enabled` is not supported: Helix's lock screen offers no
  Restart or Power Off.
