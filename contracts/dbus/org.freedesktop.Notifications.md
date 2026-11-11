# org.freedesktop.Notifications — behavioral notes

Companion to `org.freedesktop.Notifications.xml`: the freedesktop.org
Desktop Notifications specification (1.2), as GNOME Shell serves it.

## Bus name and object

- Bus name: `org.freedesktop.Notifications`
- Object: `/org/freedesktop/Notifications`

## `Notify`

- `replaces_id`: 0 for a new notification; otherwise the id of a
  notification to replace in place (same id returned). An unknown id is
  treated as 0.
- `actions`: flat list of pairs `[key, label, key, label, …]`. The key
  `default` is the action for clicking the notification itself and is not
  shown as a button.
- `expire_timeout` (ms): -1 = server default, 0 = never expires.
- Returns the notification id (never 0).

Hints (`a{sv}`) used by GNOME clients:

| Hint | Type | Meaning |
|------|------|---------|
| `urgency` | `y` | 0 low, 1 normal, 2 critical. Critical notifications do not expire. |
| `resident` | `b` | stay after an action is invoked |
| `transient` | `b` | do not keep after display |
| `category` | `s` | e.g. `email.arrived` |
| `desktop-entry` | `s` | the sending app's desktop file id, without `.desktop` |
| `image-path` / `image-data` | `s` / `(iiibiiay)` | an image |
| `suppress-sound`, `sound-file`, `sound-name` | | sounds |

## Closing

`NotificationClosed(id, reason)` is emitted whenever a notification goes
away. Reasons: 1 expired, 2 dismissed by the user, 3 closed by
`CloseNotification`, 4 undefined/reserved.

## Actions

When the user picks an action, the server emits
`ActivationToken(id, token)` (an XDG activation token the app can use to
raise its window) if it has one, then `ActionInvoked(id, key)`. The
notification is then closed (reason 2) unless it is `resident`.

## Server information and capabilities

`GetServerInformation` returns name, vendor, version and the spec version
(`1.2`). `GetCapabilities` lists optional features the server supports,
among: `actions`, `body`, `body-hyperlinks`, `body-images`,
`body-markup`, `icon-multi`, `icon-static`, `persistence`, `sound`.
Clients must not rely on features the server does not list.
