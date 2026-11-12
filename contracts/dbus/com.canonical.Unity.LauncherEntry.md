# com.canonical.Unity.LauncherEntry — behavioral notes

Apps put a count, a progress bar or an "urgent" mark on their launcher
icon by sending a signal on the session bus (Unity's launcher API, used by
Ubuntu's dock, KDE and others; supported by GTK and Qt apps such as
Firefox, Thunderbird, Steam and Telegram through libunity or their own
code). There is no well-known name or object: listeners match the signal.

- Signal `Update(s app_uri, a{sv} properties)` on interface
  `com.canonical.Unity.LauncherEntry`, from any object path.
  `app_uri` is `application://<desktop file id>` (e.g.
  `application://firefox.desktop`).
- Properties (each optional; missing ones keep their previous value):
  `count` (`x`), `count-visible` (`b`), `progress` (`d`, 0 to 1),
  `progress-visible` (`b`), `urgent` (`b`), `quicklist` (`s` or `o`: an
  object path, on the sender's connection, of a menu with the
  `com.canonical.dbusmenu` interface; empty or `/`: none).
- An app's values last until it updates them or leaves the bus.

## Helix

The shell follows the signal for the dock (`contracts/wayland/README.md`,
"Dock"). The app is the desktop file id without `.desktop`, matched
against the dock's apps as their ids are (case aside, or through the
desktop file whose `StartupWMClass` names it). `count` is also read when
sent as `i` or `u`.

- A visible count above 0 shows as the icon's red badge (9+ above nine),
  in place of the app's notification count.
- Visible progress shows as a bar across the bottom of the icon, filled
  in the accent color (values kept within 0 and 1).
- `urgent` true gives the icon an orange background until it is false.
- When the sender of an app's updates leaves the bus, its values go.

### Quicklists

An app's quicklist entries are added to its dock menu
(`contracts/wayland/README.md`, "Dock").

- `com.canonical.dbusmenu` as used: `AboutToShow(i id) → b`,
  `GetLayout(i parent, i depth, as properties) → (u revision,
  (i id, a{sv} properties, av children))`, `Event(i id, s event, v data,
  u timestamp)` and the signal `LayoutUpdated(u revision, i parent)`.
- The shell reads the layout (`AboutToShow(0)`, then `GetLayout(0, -1,
  [])`) when an app sends a new `quicklist`, and again whenever that menu
  sends `LayoutUpdated`.
- Entries are the root's children, in order: `visible` false ones are
  left out; `type` `separator` is a separator (never first, last or twice
  in a row); an entry with `children-display` `submenu` is replaced by its
  own children (one flat list); other entries show their `label` with
  mnemonic underscores removed (`__` is a `_`); entries without a label,
  or with `enabled` false, are left out; `toggle-type` `checkmark` or
  `radio` shows a check box, checked when `toggle-state` is 1.
- Choosing an entry sends it `Event(id, "clicked", "" , seconds since
  1970)`.
