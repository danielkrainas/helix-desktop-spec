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
| `urgency` | `y` | 0 low, 1 normal, 2 critical. Critical notifications do not expire (Helix: see "How long banners stay"). |
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

*GNOME does not document this; Helix's choice:* how notifications are kept
after their banner (the list, grouping, stacking, limits, `expire_timeout`
and `transient`) and the paths of the per-application settings, as below.

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

*Helix's choice:* limits. `image-data` wider or taller than 2048, with a
rowstride that is negative or shorter than a row, or with less data than
its size needs, is ignored (the next source is used); larger than 512 it
is kept shrunk to 512. Image files are read only when they are regular
files of at most 16 MiB (never devices or FIFOs), and PNGs only up to
4096 pixels a side. Summaries longer than 1000 characters and bodies
longer than 10 000 are cut, with "…".

### Body markup

The body may use the markup of the Desktop Notifications specification:
`<b>` (bold), `<i>` (italic; Helix draws it in the regular face), `<u>`
(underlined), `<a href="URI">` (a link) and the entities `&amp;`, `&lt;`,
`&gt;`, `&quot;`, `&apos;` and `&#N;` / `&#xN;`. Other tags, `<img>`
included, are left out with their text kept. A `<` that does not start a
tag is shown as is (a tag ends at the first `>`, and never spans another
`<`). Links are underlined in the accent color; clicking one
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
- `force-expanded` true: the app's banners show up to eight lines of the
  body instead of two (the list is unchanged).

### How long banners stay

*GNOME does not document this; Helix's choice:* when a banner leaves the
screen is Helix's, as follows. One banner shows at a time; the rest wait
in order.

- A banner's time counts only while the user is active: there was
  keyboard, pointer or touch input in the last 5 seconds (by the
  compositor's `org.gnome.Mutter.IdleMonitor` `GetIdletime`; without an
  idle monitor the user counts as active). A banner that arrives while
  the user is away is still there when they come back, and then counts
  as usual.
- It does not count while the pointer is over the banner or the banner
  has keyboard focus.
- A normal or low-urgency notification's banner stays for its
  `expire_timeout` (5 seconds for -1) of counted time.
- A banner with `expire_timeout` 0 (no timeout) stays for
  `banner-ignore-timeout` seconds of counted time
  (`contracts/gsettings/org.helixdesktop.shell.md`, default 10).
- A critical notification's banner ignores `expire_timeout` and stays for
  `critical-banner-ignore-timeout` seconds of counted time (default 30).
  It shows even with do not disturb on.
- With either key 0, those banners stay until the user acts on them.
  Changes to the keys apply at once.
- When its time is up the banner moves its notification to the list (see
  below); the notification is not closed, and the app is not told,
  except for `transient` notifications, which close with reason 1.
- On a monitor in full screen (its topmost window is fullscreen; see
  `contracts/wayland/README.md`, "Monitors and windows") normal and
  low-urgency banners are hidden, as GNOME holds banners back there;
  their time still counts, so they end up in the list. Critical banners
  show above the full-screen window.
- Replacing the notification on screen (`replaces_id`) starts its count
  again. When the screen locks, the banner goes back to the front of the
  queue and shows again, counting from the start, after unlocking.

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
its banner. Agents with Agent Access can dismiss a listed notification
and press its buttons the same way (`DismissNotification`,
`NotificationAction`; `contracts/agent-access/README.md`,
"Notifications, files and monitors").

Banners and list rows say when the notification arrived (or was last
replaced), after the app's name: "Chat · now". *GNOME does not document
this; Helix's choice:* the words are "now" (under a minute), "5 min"
(under an hour), "2 h" (earlier today), "Yesterday", the weekday (within
the last six days), then the date ("2 Oct", with the year when it is not
this year). They move on each minute while the list is open. On the
accessibility bus a row's description starts with the long form ("just
now", "5 minutes ago", "2 hours ago", "Yesterday", …). The time is kept
while the shell runs (the list itself is not kept across restarts).

Notifications are grouped by app (its app id, see "Per-application
settings"; without one, its app name). Groups come in the order of their
newest notification. A group of one is shown as its notification alone.
A larger group has a header row: an arrow, the app name, "<n>
notifications" and a × that dismisses all of the group's notifications
(reason 2 each); below it, only the newest notification while the group is
closed, all of them (indented) when open. Clicking the header opens or
closes the group; groups start closed each time the list opens. The list
scrolls by rows (headers and notifications). The list also has "Clear" (dismisses all) and a "Do Not
Disturb" switch (`org.gnome.desktop.notifications show-banners`, inverted).
Escape or a click outside closes it. Beside the list are the date, a
calendar, world clocks and the media player (`contracts/gsettings/calendar.md`).
While the list holds notifications
that were posted since it was last opened, the panel shows a dot next to
the clock.

On short screens the list's box is no taller than its monitor below the
panel (8 logical px kept free); the notifications scroll by rows as
before, and the calendar column (date, month, events, world clocks,
weather, media) scrolls with the wheel over it, a thin bar showing
where it is. Focusing a calendar control out of sight from the keyboard
scrolls it into sight.

Keyboard (Helix):

- Opened from the keyboard (`toggle-message-tray`, or Enter on the
  panel's clock while the panel has the keyboard), the list takes the
  keyboard (see `contracts/wayland/README.md`, "Shell surfaces and the
  keyboard") and the first row (a notification or a group's header) has
  the focus, else Do Not Disturb. Opened by a click, the first Tab or
  arrow focuses the first row.
- Tab and Shift+Tab go, wrapping, through Do Not Disturb, Clear, each row
  followed by its × and its links, then the calendar's previous and next
  month buttons, the selected day and the media buttons. Ctrl+Tab jumps
  between the notifications and the calendar.
- In the notifications: Up and Down move between rows (scrolling the
  list as needed; Up from the first row reaches Do Not Disturb), Home
  and End to the first and last row; Enter or Space acts as a click on
  what has the focus (a row activates its notification, a header opens
  or closes its group, a link opens); Delete dismisses the focused
  notification or group; Right opens a group, Left closes it (from a
  notification inside it too, the header taking the focus).
- In the calendar: on the selected day, the arrow keys move it by a day
  or a week (crossing into other months), Page Up and Page Down by a
  month (same day, or the month's last), Home to today; on the month and
  media buttons, Left and Right move along them, Down (from the month
  buttons) or Up (from the media buttons) goes to the selected day, Enter
  or Space presses them.
- Escape closes the list; the clock gets the keyboard back when the list
  was opened from it.
- The focused control has the focus ring (a ring around the selected
  day) and is the one marked focused on the accessibility bus. There the
  calendar also has the month's title, the days (the selected one
  selected) and a list of the selected day's events ("TIME: SUMMARY",
  "All Day: SUMMARY", or "No Events").
