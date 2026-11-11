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

How the modes interact with active inhibitors is still open (see `gaps.md`).
