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
- any other action (not exported on the app's `app.` action group) →
  the `ActionInvoked(app_id, id, action, parameter, platform_data)`
  signal instead, with `parameter` holding the target if there is one.
- Helix: when an `app.` action cannot be delivered because the app cannot
  be reached at its bus name (the call fails with `ServiceUnknown`,
  `NameHasNoOwner` or an unknown object or interface), the shell sends
  `ActionInvoked` for it too, with the full action name (`app.NAME`).

The notification is then removed.

gnome-shell does not document `ActionInvoked` (new in GNOME 51). Helix
gives it the meaning xdg-desktop-portal documents for the same signal of
its Notification backend interface: it is sent "to the application when a
non-exported action is activated" (see `gaps.md`).

## Helix

The `icon` (a serialized `GIcon`: themed icon names, a file, or image
bytes) is shown, else the application's own icon (its desktop file
`app_id.desktop`); the app's name is its desktop file's name, else
`app_id`. The body is plain text; a `markup-body` (`s`, notification
markup as in `org.freedesktop.Notifications.md`) is shown instead when
given (the Notification portal's key; see `org.freedesktop.impl.portal.md`). Notifications stay in the
notification list as `org.freedesktop.Notifications.md` ("The
notification list") describes, until withdrawn, activated or dismissed.
