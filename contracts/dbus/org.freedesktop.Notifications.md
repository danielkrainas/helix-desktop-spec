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

## Helix

`GetCapabilities` returns `actions`, `body`, `body-hyperlinks`,
`body-markup`, `icon-static`, `persistence` and `sound`.

### Icons

A notification shows one icon, the first of:

1. the `image-data` hint (`(iiibiiay)`: width, height, rowstride, has
   alpha, bits per sample (8), channels (3 or 4), RGB(A) data), or its
   older names `image_data` and `icon_data`;
2. the `image-path` hint (or `image_path`): an absolute path, a `file://`
   URI, or an icon name;
3. the `app_icon` argument, in the same forms;
4. the `Icon` of the desktop file named by the `desktop-entry` hint.

Icon names are looked up in the theme of `org.gnome.desktop.interface
icon-theme`, then `hicolor`; PNG and SVG icons are shown.

### Body markup

The body may use the markup of the Desktop Notifications specification:
`<b>` (bold), `<i>` (italic; Helix draws it in the regular face), `<u>`
(underlined), `<a href="URI">` (a link) and the entities `&amp;`, `&lt;`,
`&gt;`, `&quot;`, `&apos;` and `&#N;` / `&#xN;`. Other tags, `<img>`
included, are left out with their text kept. A `<` that does not start a
tag is shown as is. Links are underlined in the accent color; clicking one
opens its URI with the default application (and does not activate the
notification).

### Sounds

A notification plays a sound once, when it is posted: the `sound-file`
hint (a path), else the `sound-name` hint (a name from the freedesktop
sound theme specification, looked up in the theme of
`org.gnome.desktop.sound theme-name`, then `freedesktop`). Nothing plays
with the `suppress-sound` hint, with `org.gnome.desktop.sound
event-sounds` false, or when do not disturb hides the banner.

### Per-application settings

GNOME Settings keeps per-app notification settings in the relocatable
schema `org.gnome.desktop.notifications.application` (key table in
`contracts/gsettings/org.gnome.desktop.notifications.application.md`; keys `application-id`,
`enable`, `show-banners`, `enable-sound-alerts`, `force-expanded`,
`show-in-lock-screen`, `details-in-lock-screen`), one copy per app, and
lists the apps named in `org.gnome.desktop.notifications`
`application-children`.

- A notification's app is the GTK notification's app id, or the
  `desktop-entry` hint of a freedesktop notification (without a trailing
  `.desktop`). Notifications without either (and the shell's own) have no
  per-app settings.
- The app's settings live at
  `/org/gnome/desktop/notifications/application/<child>/`, `<child>` being
  the app id in lower case with every character other than `a`–`z`,
  `0`–`9` and `-` turned into `-` (`org.gnome.Calendar` →
  `org-gnome-calendar`). The first time an app notifies, the shell adds
  `<child>` to `application-children` and sets its `application-id` to
  `<app id>.desktop`, so GNOME Settings lists the app.
- `enable` false: the notification is dropped: `Notify` still returns an
  id, nothing is shown, played or listed, and no signal follows.
- `show-banners` false: no banner (and no sound); the notification goes
  straight to the list, as with do not disturb.
- `enable-sound-alerts` false: no sound.
- Critical notifications get a banner even when banners are off (as with
  do not disturb), but not when `enable` is false.
- `show-in-lock-screen` false: the app's notifications are left out of
  the lock screen; `details-in-lock-screen` true: the lock screen shows
  the text of its newest ones (`org.gnome.ScreenSaver.md`, "Lock screen
  (Helix)"). Both only matter while the global `show-in-lock-screen` is
  true. Apps without settings show counts only.
- `force-expanded` has no effect (Helix's banners always show the body).

### The notification list (persistence)

Notifications stay in a list after their banner goes, until the user
dismisses or activates them or the app closes them (`CloseNotification`,
`RemoveNotification`). A banner timing out moves its notification to the
list without closing it; only a notification with the `transient` hint is
closed (reason 1) when its banner times out. Do not disturb (`show-banners`
false) sends notifications straight to the list. At most 50 are kept; the
oldest beyond that is closed (reason 1).

Clicking the clock in the panel opens the list below it, newest first:
each notification's icon, app, summary and up to two lines of body, with a
× that dismisses it (reason 2). Clicking a notification acts as clicking
its banner. The list also has "Clear" (dismisses all) and a "Do Not
Disturb" switch (`org.gnome.desktop.notifications show-banners`, inverted).
Escape or a click outside closes it. Beside the list are the date, a
calendar, world clocks and the media player (`contracts/gsettings/calendar.md`).
While the list holds notifications
that were posted since it was last opened, the panel shows a dot next to
the clock.
