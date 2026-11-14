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
  before its unlock dialog. Helix's lock screen rests first (the time,
  the date and the cards; see `contracts/dbus/org.gnome.ScreenSaver.md`,
  "Lock screen (Helix)") and shows the prompt on a key or click; the
  prompt always shows who is locked out (as GNOME's unlock dialog does):
  the user's picture and name above the password field. Helix ignores
  the key.
  - The name is AccountsService's `RealName` for the user
    (`org.freedesktop.Accounts.User`, found with `FindUserByName`, read
    only), else the real name in the GECOS field of the password
    database (its first comma-separated part), else the login name. It
    is drawn in the foreground color, bold, 22 logical px; the label on
    the accessibility bus is the same string.
  - The picture is AccountsService's `IconFile` (PNG, JPEG, WebP, JPEG
    XL or SVG, at most 8 MiB), cropped to a circle with an anti-aliased
    edge. Without one: the name's initials (first letters of its first
    and last words) in white on a color picked from the name, the same
    for a name every time; in high contrast, on black with a white ring.
    The login screen draws the same avatar.
  - Both are read when the shell starts and each time the screen locks,
    never while drawing.
- `restart-enabled`: with it on, the lock screen has "Restart…" and
  "Power Off…" buttons at its bottom left (each only when logind allows
  it). Choosing one asks, in a card in the middle of the screen over a
  dimmed lock screen (which no longer reacts): "Restart this computer?" /
  "Power off this computer?", with "Cancel" and "Restart" / "Power Off";
  confirming asks the session manager to reboot or shut down (`Reboot` /
  `Shutdown`), and the shell's end-session dialog answers that request as
  confirmed without showing. While asking, apps holding up logout (the
  session manager's inhibitors with the logout flag) are listed in the
  card under "Some apps are busy:" (the app in bold, its reason under
  it, at most four), and the question becomes "Restart anyway?" / "Power
  off anyway?"; confirming goes ahead without waiting for them. Read each
  time the lock screen is drawn.
- The bottom buttons are on the monitor with the prompt only
  (`contracts/dbus/org.gnome.ScreenSaver.md`, "Monitors").
- The bottom buttons ("Restart…", "Power Off…", "Switch User…") are round
  icon buttons, 48 logical px (system-reboot, system-shutdown,
  system-switch-user symbolic icons), captioned with their name, with a
  light translucent fill that brightens when pointed at and when pressed
  (in high contrast black with a white border); their targets are at
  least 44 px. In the question, "Cancel" is a light button and the
  confirming button is red (#C01C28, white text at 4.5:1 or more), never
  the accent; in high contrast both are black with a white border and are
  told apart by their words.
- The keyboard reaches them: Tab and Shift+Tab move between the password
  field, the media buttons and the bottom buttons, with an accent ring
  around the focused one; Return or Space presses it; typing goes back to
  the field. In the question the focus starts on "Cancel", Tab or ← → move
  between its two buttons, and Escape cancels.
