# org.gnome.SessionManager — behavioral notes

Companion to `org.gnome.SessionManager.xml`. Records value semantics that the
introspection XML cannot express.

## `Logout(u mode)`

| Value | Name            | Meaning                                         |
|-------|-----------------|-------------------------------------------------|
| 0     | normal          | Log out, showing a confirmation if needed       |
| 1     | no confirmation | Log out without asking the user to confirm      |
| 2     | force           | Log out regardless of confirmation              |

These are the three values in use by existing GNOME clients. No other values
are defined; implementations should reject any other value with a D-Bus error
rather than guessing.

## Inhibit flags (`Inhibit`, `IsInhibited`, `InhibitedActions`)

`flags` is a bitmask. These are the bits in use by existing GNOME clients and
by `org.freedesktop.portal.Inhibit` (which uses the same values):

| Bit | Value | Inhibits                  |
|-----|-------|---------------------------|
| 0   | 1     | logout (and shutdown / reboot) |
| 1   | 2     | user switching            |
| 2   | 4     | suspend                   |
| 3   | 8     | the session being marked idle |
| 4   | 16    | auto-mounting of media    |

Unknown bits are carried through unchanged (they still count for
`IsInhibited` and appear in `InhibitedActions`), but carry no meaning.

`IsInhibited(flags)` is true when any active inhibitor shares a bit with
`flags`. `InhibitedActions` is the bitwise OR of all active inhibitors' flags.

## Ending the session and inhibitors

- `Logout(0)` and `Logout(1)` are blocked while any inhibitor has the logout
  bit (1). With a confirmation UI the user may choose to override; without one
  the call fails.
- `Logout(2)` (force) ignores inhibitors.
- `Shutdown` and `Reboot` end the session like `Logout(0)` and are blocked by
  the logout bit in the same way.
- Inhibitors without the logout bit (e.g. idle-only) never block ending the
  session.

## `GetLocale(i category)`

`category` is the platform C library's `LC_*` constant (e.g. `LC_MESSAGES`).
The reply is the session manager's current locale for that category, as the C
library reports it. An unknown category is an error.

## `InitializationError(s message, b fatal)`

Only valid during the initialization phase (before `Initialized`). Reports a
failure starting a session component. If `fatal` is true the session ends.

## `RestoreSupported`

When false, clients must not use `RegisterRestore`, `DeletedInstanceIds` or
`UnregisterRestore`.
