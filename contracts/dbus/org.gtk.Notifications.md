# org.gtk.Notifications — behavioral notes

Companion to `org.gtk.Notifications.xml`. GLib's `GNotification` uses this
interface when its name has an owner (GNOME Shell serves it), falling back
to `org.freedesktop.Notifications` otherwise.

## Bus name and object

- Bus name: `org.gtk.Notifications`
- Object: `/org/gtk/Notifications`

## `AddNotification(app_id, id, notification)`

Adds or replaces (same `app_id` + `id`) a notification. `app_id` is the
application id (also its bus name and desktop file id). `notification`
keys (GLib's serialization, also used by the XDG Notification portal):

| Key | Type | Meaning |
|-----|------|---------|
| `title` | `s` | |
| `body` | `s` | |
| `icon` | `v` | serialized `GIcon` |
| `priority` | `s` | `low`, `normal`, `high`, `urgent` (urgent does not expire) |
| `default-action` | `s` | detailed action name, e.g. `app.open` |
| `default-action-target` | `v` | its parameter |
| `buttons` | `aa{sv}` | each `label` (`s`), `action` (`s`), optional `target` (`v`) |

## `RemoveNotification(app_id, id)`

Withdraws it.

## Activation

Clicking the notification or a button activates the app over
`org.freedesktop.Application` (see `org.freedesktop.Application.md`) at bus
name `app_id`, object path derived from it (`org.gnome.Calculator` →
`/org/gnome/Calculator`):

- an `app.NAME` action → `ActivateAction("NAME", [target] or [], platform_data)`
- no default action → `Activate(platform_data)`

The notification is then removed.

The meaning of the `ActionInvoked` signal (new in GNOME 51) is not
documented; see `gaps.md`.
