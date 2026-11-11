# XDG desktop portal backends — behavioral notes

Companion to the `org.freedesktop.impl.portal.*.xml` contracts, pinned
against xdg-desktop-portal 1.22.1. These are the *backend* interfaces: the
xdg-desktop-portal frontend calls them on behalf of applications.

## Registration

A backend owns a bus name and serves its interfaces at
`/org/freedesktop/portal/desktop`. It is advertised to the frontend with a
`<name>.portal` file in `$datadir/xdg-desktop-portal/portals/`:

```ini
[portal]
DBusName=<bus name>
Interfaces=<interface>;<interface>;
```

The frontend picks backends per interface from
`$datadir/xdg-desktop-portal/<desktop>-portals.conf`, where `<desktop>` is
each lowercased entry of `XDG_CURRENT_DESKTOP` in order:

```ini
[preferred]
default=<backend>;<fallback>;
```

## Request (`org.freedesktop.impl.portal.Request`)

Methods that take a `handle` export a Request object at that path for the
duration of the interaction. The frontend calls `Close` to end it, after
which the object must be removed.

## Session (`org.freedesktop.impl.portal.Session`)

Methods that take a `session_handle` export a Session object at that path.
`Close` ends the session; `Closed` is emitted when the backend ends it.

## Settings

`ReadAll(namespaces)`: an empty list, or one containing an empty string,
matches all namespaces. An entry ending in `.*` matches namespaces with that
prefix. Otherwise matching is exact.

`Read(namespace, key)` fails for an unknown namespace or key. The frontend
expects the error `org.freedesktop.portal.Error.NotFound`.

`SettingChanged` is emitted whenever a value changes.

Documented keys, all in namespace `org.freedesktop.appearance`:

| Key | Type | Values |
|-----|------|--------|
| `color-scheme` | `u` | 0 no preference, 1 prefer dark, 2 prefer light |
| `accent-color` | `(ddd)` | sRGB in [0, 1]; out of range means unset |
| `contrast` | `u` | 0 no preference, 1 higher contrast |
| `reduced-motion` | `u` | 0 no preference, 1 reduced motion |

Unknown values are treated as 0. Backends may expose other namespaces; they
are undocumented (see `gaps.md`).

### Helix

`org.freedesktop.appearance` is derived from GSettings:

- `color-scheme` from `org.gnome.desktop.interface` `color-scheme`
  (`default` 0, `prefer-dark` 1, `prefer-light` 2);
- `contrast` 1 when `org.gnome.desktop.a11y.interface` `high-contrast` is
  true;
- `reduced-motion` 1 when `org.gnome.desktop.a11y.interface`
  `reduced-motion` is `reduce` (on schemas without that key: when
  `org.gnome.desktop.interface` `enable-animations` is false);
- `accent-color` from `org.gnome.desktop.interface` `accent-color`, using
  the colors libadwaita documents for its accent colors
  (`AdwAccentColor`), as sRGB values (each hex pair divided by 255):

  | Name | Color |
  |------|-------|
  | `blue` | `#3584e4` |
  | `teal` | `#2190a4` |
  | `green` | `#3a944a` |
  | `yellow` | `#c88800` |
  | `orange` | `#ed5b00` |
  | `red` | `#e62d42` |
  | `pink` | `#d56199` |
  | `purple` | `#9141ac` |
  | `slate` | `#6f8396` |

  On schemas without the key, or with a name not in the table, the value
  is `(-1, -1, -1)` (out of range: unset).

The raw GSettings schemas are exposed as namespaces of the same name, with
every key: `org.gnome.desktop.interface`, `org.gnome.desktop.a11y.interface`,
`org.gnome.desktop.wm.preferences` (window button layout and the like),
and the app-facing `org.gnome.desktop.sound` (sound theme, event and input
feedback sounds), `org.gnome.desktop.privacy` (recent files) and
`org.gnome.desktop.peripherals.mouse` (double-click time, drag threshold),
so sandboxed apps see the same settings as others. Schemas that are not
installed are left out. `SettingChanged` follows every key.

## Inhibit

`Inhibit(handle, app_id, window, flags, options)` inhibits until `Close` is
called on the Request at `handle`. Flags: 1 logout, 2 user switch, 4 suspend,
8 idle — the same bits as `org.gnome.SessionManager` inhibitors.
`options["reason"]` (`s`) is a user-visible reason.

`CreateMonitor` exports a Session at `session_handle`, returns `0` on success,
and sends `StateChanged(session_handle, state)` while the session is open.
`state` keys:

| Key | Type | Values |
|-----|------|--------|
| `screensaver-active` | `b` | whether the screensaver is active |
| `session-state` | `u` | 1 running, 2 query end, 3 ending |

On `session-state` 2, monitors must call `QueryEndResponse(session_handle)`
within one second.

## ScreenCast

`AvailableSourceTypes`: 1 monitor, 2 window, 4 virtual.
`AvailableCursorModes`: 1 hidden, 2 embedded, 4 metadata.

- `CreateSession(handle, session_handle, app_id, options)` exports a
  Session at `session_handle`; `results["session_id"]` (`s`).
- `SelectSources(handle, session_handle, app_id, options)`: `types` (`u`,
  default monitor), `multiple` (`b`), `cursor_mode` (`u`, one of the
  available ones, default hidden; another closes the session),
  `persist_mode` (`u`: 0 none, 1 while the app runs, 2 until revoked) and
  `restore_data` (`(suv)`).
- `Start(handle, session_handle, app_id, parent_window, options)`
  typically asks the user, then returns `results["streams"]`
  (`a(ua{sv})`): per stream its PipeWire node id and `position` (`(ii)`),
  `size` (`(ii)`), `source_type` (`u`) and optionally `mapping_id` (`s`)
  and `pipewire-serial` (`t`); and optionally `persist_mode` and
  `restore_data`.
- Closing the Session ends the screen cast.

### Helix

- `AvailableSourceTypes` 3 (monitors, windows), `AvailableCursorModes` 3
  (hidden, embedded), `version` 4.
- `SelectSources` answers 2 when `types` has neither monitors nor windows,
  or `cursor_mode` is not 1 or 2 (closing the session). `multiple` is
  accepted, but one source is shared.
- `Start` asks the user through the shell
  (`org.helix.ScreenCastChooser.md`), offering the types asked for, and
  answers 1 when they cancel. Then it records the choice with
  `org.gnome.Mutter.ScreenCast` (`org.gnome.Mutter.ScreenCast.md`) and
  returns one stream with `size` and `source_type` (and `position` for a
  monitor).
- Remembering: when `persist_mode` is 1 or 2 and a monitor is shared,
  `Start` returns the same `persist_mode` and `restore_data`
  `("Helix", 1, <a{sv}>)` with the monitor's `connector`, `vendor`,
  `product` and `serial` (all `s`). Given such `restore_data` in
  `SelectSources`, `Start` shares that monitor without asking if a monitor
  with the same vendor, product and serial is on (else the one on that
  connector, if its identity is unknown on both sides), and asks
  otherwise. Shared windows are not remembered (`persist_mode` 0).
- When the compositor ends the recording, the Session emits `Closed`.

## Screenshot

Methods that take a `handle` answer with `response`: 0 success, 1
cancelled by the user, 2 ended another way (failed, refused). Version 3 of
the interface.

`AvailableTargets` is a bitmask: 1 the whole screen, 2 a window the user
picks, 4 an area the user selects, 8 the active window.

`Screenshot(handle, app_id, parent_window, options)` returns
`results["uri"]` (`s`), the URI of the image. `options`:

| Key | Type | Meaning |
|-----|------|---------|
| `modal` | `b` | the dialog is modal; default true |
| `interactive` | `b` | let the user adjust the screenshot first; default false |
| `target` | `u` | one value from `AvailableTargets`; absent means the behavior before version 3 (the whole screen) |
| `permission_store_checked` | `b` | the frontend already checked the app's `screenshot` permission; default false |

`PickColor(handle, app_id, parent_window, options)` returns
`results["color"]` (`(ddd)`), sRGB values from 0 to 1.

### Helix

- `AvailableTargets` is 13 (screen, area, active window); `version` is 3.
- With `interactive` true, or `permission_store_checked` false (nobody
  asked the user yet), the shell's screenshot UI is shown
  (`org.gnome.Shell.Screenshot.md`), so the user decides what, if
  anything, is captured.
- Otherwise `target` decides: 1 or absent the whole screen, 4 an area the
  user selects, 8 the focused window. A target that is not available
  answers 2.
- Images are saved like the screenshot UI's: in the `Screenshots` folder of
  the pictures directory, named `Screenshot From YYYY-MM-DD HH-MM-SS.png`.
- `PickColor` uses the shell's color picker. Cancelling answers 1.
- The portal calls `org.gnome.Shell.Screenshot`, which is why its bus name
  is on that interface's allowed list.
