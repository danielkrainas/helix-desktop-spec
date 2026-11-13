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
- `disable-show-password`: the password fields of the shell's dialogs
  (polkit, keyrings, networks, mounts) have no "Show" button. With it
  off, "Show" beside a field shows its text, and "Hide" dots again. The
  lock screen's field has no such button either way. Read when a dialog
  opens.
- `disable-user-switching`: no "Switch User…" (below), and a request to
  switch is refused.
- `mount-removable-storage-devices-as-read-only`: Helix mounts drives
  through GIO, whose GVfs UDisks2 volume monitor reads this key and
  mounts removable drives read-only; Helix does nothing more. (Not tried
  with a real drive.)
- The other keys (`disable-command-line`, `user-administration-disabled`,
  …) have no effect in the shell: it has no run-command dialog or user
  administration; GNOME Settings reads the latter itself.

## User switching

"Switch User…" is in the Power menu, after "Lock Screen", when:

- Helix runs on bare hardware (never nested: it would drive the host's
  display manager);
- a display manager that can show its login screen runs on the system
  bus: GDM (`org.gnome.DisplayManager`, whose
  `/org/gnome/DisplayManager/LocalDisplayFactory`
  `org.gnome.DisplayManager.LocalDisplayFactory.CreateTransientDisplay()
  → o` shows a new login screen) is used first, else LightDM
  (`org.freedesktop.DisplayManager`, the session's seat object named by
  `XDG_SEAT_PATH`, `org.freedesktop.DisplayManager.Seat.SwitchToGreeter()`);
- AccountsService (`org.freedesktop.Accounts` `ListCachedUsers`) lists
  more than one user (or AccountsService is not running);
- `disable-user-switching` is off.

Choosing it locks the session first (as "Lock Screen" does) and, once the
lock screen shows (within 3 seconds), asks the display manager; when the
session cannot be locked, nothing is switched. Whether to offer it is
decided when the shell starts.
