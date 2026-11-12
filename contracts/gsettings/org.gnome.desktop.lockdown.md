# `org.gnome.desktop.lockdown`

Key surface of the `org.gnome.desktop.lockdown` GSettings schema in gsettings-desktop-schemas
51.0, described through GSettings introspection of the compiled schema.

Path: `/org/gnome/desktop/lockdown/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `disable-application-handlers` | `b` | `false` |  |
| `disable-command-line` | `b` | `false` |  |
| `disable-lock-screen` | `b` | `false` |  |
| `disable-log-out` | `b` | `false` |  |
| `disable-printing` | `b` | `false` |  |
| `disable-print-setup` | `b` | `false` |  |
| `disable-save-to-disk` | `b` | `false` |  |
| `disable-show-password` | `b` | `false` |  |
| `disable-user-switching` | `b` | `false` |  |
| `mount-removable-storage-devices-as-read-only` | `b` | `false` |  |
| `user-administration-disabled` | `b` | `false` |  |

## Helix

- `disable-lock-screen`: the shell does not lock (requests to lock fail,
  idle locking does not happen) and its Power menu has no Lock Screen.
- `disable-log-out`: the Power menu has no Log Out, and the `logout`
  key does nothing.
- `disable-printing`, `disable-save-to-disk`,
  `disable-application-handlers`: passed to apps through the Lockdown
  portal (`contracts/dbus/org.freedesktop.impl.portal.md`, "Lockdown");
  Helix's own print and file dialogs honor the first two.
- The other keys have no effect in Helix (it has no run-command dialog,
  user switching or password entries with a "Show Password" item, and
  mounts removable media as they come).
