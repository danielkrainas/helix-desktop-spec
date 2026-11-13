# Status icons (StatusNotifierItem) — behavioral notes

Companion to `org.kde.StatusNotifierWatcher.xml`,
`org.kde.StatusNotifierItem.xml` and `com.canonical.dbusmenu.xml`. Apps
show status icons ("tray icons") through the freedesktop.org Status
Notifier Item specification: libappindicator and
libayatana-appindicator, Qt's `QSystemTrayIcon`, Electron and others.
The specification writes the names as `org.freedesktop.*`, but apps use
`org.kde.*`, and so does Helix.

The shell is both the **watcher** (it keeps the list of items) and the
**host** (it shows them), in the panel item `helix:tray`
(`contracts/wayland/README.md`, "Panel layout").

## Watcher

- Bus name `org.kde.StatusNotifierWatcher`, object
  `/StatusNotifierWatcher`, interface `org.kde.StatusNotifierWatcher`.
  If another process holds the name, Helix shows no status icons (and
  logs it).
- `RegisterStatusNotifierItem(service)`:
  - `service` starting with `/` is an object path on the caller's
    connection; libayatana-appindicator does this (observed:
    `/org/ayatana/NotificationItem/<id>`).
  - Otherwise it is a bus name (Qt: `org.kde.StatusNotifierItem-<pid>-<n>`),
    with the item at `/StatusNotifierItem`. A bus name with no owner is
    an `org.freedesktop.DBus.Error.InvalidArgs` error.
  - An item is identified by its connection's unique name and its path.
    Registering the same item again changes nothing.
  - At most 64 items; beyond that the call fails with
    `org.freedesktop.DBus.Error.LimitsExceeded`.
- `RegisteredStatusNotifierItems` lists the items as
  `<unique name><path>` (for example `:1.42/org/ayatana/NotificationItem/x`),
  in the order they registered.
- An item goes away when its connection leaves the bus
  (`StatusNotifierItemUnregistered`). Items cannot unregister otherwise.
- `RegisterStatusNotifierHost(service)` is accepted and has no effect.
  `IsStatusNotifierHostRegistered` is true (the shell is a host);
  `ProtocolVersion` is 0.

## Host: showing items

Each item's properties are read when it registers, and again after its
`New…` signals:

- `NewTitle` → `Title`;
- `NewIcon` → `IconName`, `IconPixmap`, `IconThemePath`;
- `NewAttentionIcon` → the attention icons;
- `NewOverlayIcon` → the overlay icons;
- `NewStatus` → its argument.

Missing properties count as empty.

- **Status.**
  - `Passive` items are not shown.
  - `Active` items are shown.
  - `NeedsAttention` items are shown with their attention icon (when
    they have one) and an accent-colored line under them.
- **Order.** Items are shown left to right in the order they registered.
- **Icon.**
  - The icon is `IconName` looked up first in `IconThemePath` (when not
    empty), then in the icon theme like the panel's other icons.
  - Failing that, it is `IconPixmap`: of its images, the smallest that
    is at least as large as the icon is drawn (else the largest).
    Pixmaps are ARGB32, in network byte order. Images larger than
    256×256, or whose data length does not match, are ignored.
  - Without any icon, the item is not shown.
  - Status icons are drawn 16 pixels (scaled with the monitor) and 8
    pixels apart.
- **Overlay.** `OverlayIconName` (looked up like the icon), else
  `OverlayIconPixmap`, is drawn at half the icon's size over its
  lower-right corner.
- **Tooltip.** Resting the pointer on an icon for 0.6 seconds shows a box
  below the panel, under the icon. Its first line, in bold, is
  `ToolTip`'s title, else the item's `Title`. Below it come the lines of
  `ToolTip`'s description, with its markup removed (tags dropped,
  entities decoded), at most 6 lines. Without a title or a description
  there is no tooltip. Its icon is not used. The box goes away when the
  pointer leaves the icon or clicks. It follows `NewToolTip` while shown.
  Pointer input passes through the box.
- `AttentionMovieName` and `WindowId` are not used.

## Host: clicks

- **Primary click.**
  - With `ItemIsMenu` true and a `Menu`, the menu opens.
  - Otherwise Helix calls `Activate(x, y)`, where x, y are the icon's
    bottom-left corner in the layout's coordinates. If that call fails
    (for example `UnknownMethod`: libayatana-appindicator implements
    no activation) and the item has a `Menu`, the menu opens.
- **Secondary (right) click.** The menu opens if the item has one;
  otherwise Helix calls `ContextMenu(x, y)`.
- **Middle click.** `SecondaryActivate(x, y)`.
- **Scrolling.** `Scroll(delta, orientation)`, `orientation` `vertical`
  or `horizontal`. `delta` is 120 per wheel step, negative for up or
  left, as Qt counts wheel steps.
- Every call has a 2-second deadline; Helix does not wait for it.

## Host: menus (`com.canonical.dbusmenu`)

When an item's menu opens, Helix:

1. calls `AboutToShow(0)` on the item's `Menu` object;
2. calls `GetLayout(0, -1, [])`;
3. shows the menu below the icon, like the panel's other menus (the
   same look and keys).

Each entry's properties:

- `visible` false: left out.
- `type` `separator`: a separator.
- `label`: its text, with `_` handled as the specification says.
  `__` shows one underscore; other underscores are not shown, and the
  access key they mark is not used.
- `enabled` false: shown dimmed, and does nothing.
- `toggle-type` `checkmark`: a check box, checked with `toggle-state`
  1. `radio` with `toggle-state` 1: a dot before the label.
- `children-display` `submenu`, or any children: a submenu, opened as
  the panel's other submenus are. Depth is limited to 3 levels and 100
  entries per level.
- Not used: `icon-name`, `icon-data`, `shortcut`, `disposition`.

Choosing an entry calls `Event(id, "clicked", <int32 0>, <time>)`
(the time in seconds) and closes the menu. Choosing a dimmed entry does
nothing, and the menu stays open. Helix sends no `opened` or `closed`
events; `AboutToShow` tells the app the menu is about to be shown. `LayoutUpdated` and `ItemsPropertiesUpdated` while the
menu is open are not followed; the menu is read again the next time it
opens. Menus are limited to what fits the panel's monitor.
