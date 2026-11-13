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

### Helix

Helix's backend (`helix-portals`) owns `org.freedesktop.impl.portal.desktop.helix`
and lists its interfaces in `helix.portal`. It exits when its connection to
the session bus closes, and on SIGTERM leaves the bus for at most two
seconds (a dialog still waiting for the user does not hold it up).

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

- `AvailableSourceTypes` 7 (monitors, windows, virtual),
  `AvailableCursorModes` 7 (hidden, embedded, metadata), `version` 4.
- `SelectSources` answers 2 when `types` has none of these, or
  `cursor_mode` is not 1, 2 or 4 (closing the session).
- When `types` is only virtual (4), `Start` shares a new virtual monitor
  (`RecordVirtual`) without asking: nothing that exists is shown. Its
  stream has `position`, `size` and `source_type` 4. When virtual comes
  with other types, the chooser offers "New virtual monitor" beside the
  monitors and windows.
- With `multiple`, the user may choose several sources; `Start` records
  them all in one compositor session and returns one stream per source,
  in the order chosen. Without it, one source.
- `Start` asks the user through the shell
  (`org.helix.ScreenCastChooser.md`), offering the types asked for, and
  answers 1 when they cancel. Then it records the choice with
  `org.gnome.Mutter.ScreenCast` (`org.gnome.Mutter.ScreenCast.md`) and
  returns the streams, each with `size` and `source_type` (and `position`
  for a monitor or virtual monitor).
- Remembering: when `persist_mode` is 1 or 2 and one monitor is shared
  (several sources are not remembered: the user is asked again),
  `Start` returns the same `persist_mode` and `restore_data`
  `("Helix", 1, <a{sv}>)` with the monitor's `connector`, `vendor`,
  `product` and `serial` (all `s`). Given such `restore_data` in
  `SelectSources`, `Start` shares that monitor without asking if a monitor
  with the same vendor, product and serial is on (else the one on that
  connector, if its identity is unknown on both sides), and asks
  otherwise.
- A shared window is remembered the same way, by its app id and title:
  `restore_data` `("Helix", 1, <a{sv}>)` with `window-app-id` and
  `window-title` (`s`). Restoring shares the open window with that app id
  and title (the most recently used one when several match), and asks
  when there is none.
- When the compositor ends the recording, the Session emits `Closed`.

## RemoteDesktop

`AvailableDeviceTypes`: 1 keyboard, 2 pointer, 4 touchscreen.

- `CreateSession(handle, session_handle, app_id, options)` exports a
  Session. A RemoteDesktop session may also get screen cast sources:
  `org.freedesktop.impl.portal.ScreenCast.SelectSources` on the same
  session handle.
- `SelectDevices(handle, session_handle, app_id, options)`: `types`
  (`u`, default all), `restore_data` (`(suv)`), `persist_mode` (`u`).
- `Start(handle, session_handle, app_id, parent_window, options)`
  typically asks the user; `results`: `devices` (`u`, the device types
  allowed), `clipboard_enabled` (`b`), `streams` (as for ScreenCast, when
  sources were selected) and optionally `persist_mode` / `restore_data`.
- `Notify*` inject input (see `org.gnome.Mutter.RemoteDesktop.md`); their
  stream arguments are PipeWire node ids of the session's streams, button
  and key states are `u` (0 released, 1 pressed), and
  `NotifyPointerAxis` takes `finish` (`b`) in `options`. Each may only be
  used for an allowed device type.
- `ConnectToEIS(session_handle, app_id, options)` (version 2) returns a
  socket to an EIS server for the session.

### Helix

- `AvailableDeviceTypes` 7, `version` 2.
- `Start` asks the user through the shell: with screen cast sources
  selected, the share dialog (`org.helix.ScreenCastChooser.md`) also says
  which devices the app will control; without, a dialog asks whether to
  let the app control the keyboard / pointer / touchscreen ("Allow" /
  "Deny"). Cancelling answers 1. Then it creates an
  `org.gnome.Mutter.RemoteDesktop` session (and, with sources, a screen
  cast session belonging to it), starts it, and returns `devices` (the
  types asked for, all when not given), `clipboard_enabled` and the
  streams.
- `Notify*` for a device type not allowed, before `Start`, or with an
  unknown stream are ignored. They are passed on to the compositor's
  session (node ids mapped to its stream objects).
- `ConnectToEIS` passes on the compositor's EIS socket, offering the
  allowed device types; before `Start` it fails
  (`org.freedesktop.DBus.Error.Failed`).
- Remembering: with `persist_mode` 1 or 2, `Start` returns the same
  `persist_mode` and `restore_data` `("Helix", 1, <a{sv}>)` holding
  `devices` (`u`, the device types allowed) and, when the screen was
  shared, the monitor or window keys as for ScreenCast. Given such
  `restore_data` in `SelectDevices`, `Start` does not ask when every
  device type asked for is among the remembered ones and the remembered
  monitor or window (if sources were selected) is found as for ScreenCast;
  otherwise it asks. Restore data from a ScreenCast-only session (no
  `devices`) is not enough to skip asking.
- Closing the Session ends the remote desktop session (and its screen
  cast); when the compositor ends it, the Session emits `Closed`.

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

- `AvailableTargets` is 15 (screen, window, area, active window);
  `version` is 3.
- With `interactive` true, or `permission_store_checked` false (nobody
  asked the user yet), the shell's screenshot UI is shown
  (`org.gnome.Shell.Screenshot.md`), so the user decides what, if
  anything, is captured.
- Otherwise `target` decides: 1 or absent the whole screen, 2 a window
  the user picks, 4 an area the user selects, 8 the focused window. A
  target that is not available answers 2.
- Picking a window: every monitor is dimmed with the hint "Click a window
  to capture it"; clicking a window captures it as `ScreenshotWindow`
  with `include_frame` true does (`org.gnome.Shell.Screenshot.md`).
  Clicking where there is no window does nothing; Escape or the secondary
  button cancels (answer 1).
- Images are saved like the screenshot UI's: in the `Screenshots` folder of
  the pictures directory, named `Screenshot From YYYY-MM-DD HH-MM-SS.png`.
- `PickColor` uses the shell's color picker. Cancelling answers 1.
- The portal calls `org.gnome.Shell.Screenshot`, which is why its bus name
  is on that interface's allowed list.

## Clipboard (`org.freedesktop.impl.portal.Clipboard`)

Contract: `org.freedesktop.impl.portal.Clipboard.xml`. A portal session of
another portal (RemoteDesktop, or InputCapture since its version 2) asks for the clipboard with
`RequestClipboard(session_handle, options)` before it starts; once
started, `SetSelection`, `SelectionWrite`, `SelectionWriteDone` and
`SelectionRead` take the session handle, and `SelectionOwnerChanged`
(`mime_types` `as`, `session_is_owner` `b`) and `SelectionTransfer` are
emitted with it. `version` is 1.

### Helix

- RemoteDesktop and InputCapture sessions (InputCapture: made with
  `CreateSession2`): `RequestClipboard` for any other or unknown session
  fails with `org.freedesktop.DBus.Error.InvalidArgs`; after `Start` with
  `org.freedesktop.DBus.Error.Failed`.
- No separate question: allowing the session (the RemoteDesktop `Start`
  dialog) allows the clipboard it asked for. `Start` then turns on the
  compositor session's clipboard (`EnableClipboard`, see
  `org.gnome.Mutter.RemoteDesktop.md`) and returns `clipboard_enabled`
  true (false when not asked for, or when that fails).
- The methods and signals are those of the compositor session, passed
  through (option names with `_` instead of `-`); for a session that does
  not share the clipboard the methods fail with
  `org.freedesktop.DBus.Error.Failed`. An InputCapture session's
  compositor session is its `org.helix.InputCapture` session, which
  shares the clipboard the same way (`org.helix.InputCapture.xml`); its
  `Start` turns sharing on and returns `clipboard_enabled` likewise. The
  compositor starts reporting the clipboard's owner to it from then on,
  whether or not input is being captured.

## Access (`org.freedesktop.impl.portal.Access`)

`AccessDialog(handle, app_id, parent_window, title, subtitle, body,
options) -> (response, results)`: the frontend's generic permission
question (used, for example, before letting an app use the camera).
Options: `modal` (`b`), `deny_label` and `grant_label` (`s`), `icon`
(`s`), `choices` (`a(ssa(ss)s)`: id, label, options as (id, label), and
the initial value; a choice without options is a check box, its value
`"true"` or `"false"`). Response 0 granted, 1 denied, 2 other; results:
`choices` (`a(ss)`, id and value).

### Helix

The shell's dialog (`org.helix.PortalDialog.Ask`): title the `title`,
message the `subtitle`, description the `body`, buttons `grant_label`
(default "Allow") and `deny_label` (default "Deny"). The first check-box
choice is shown as the dialog's check box; every other choice is answered
with its initial value. Response 0 when allowed, 1 when denied or closed.

## Account (`org.freedesktop.impl.portal.Account`)

`GetUserInformation(handle, app_id, window, options) -> (response,
results)`: options `reason` (`s`); results `id` (user name), `name` (real
name) and `image` (a `file://` URI of the user's picture).

### Helix

A dialog "Share Details" with "<app> wants to know your name." (`app` the
app's name from its desktop file, else its id), the description the
`reason` when given, then "Name: <real name> (<user name>)", buttons
"Share" and "Cancel". The real name is the GECOS name of the password
database (up to the first comma), else the user name; the picture is
AccountsService's `IconFile` for the user when that file exists, else
`~/.face` when it exists, else `image` is left out.

## AppChooser (`org.freedesktop.impl.portal.AppChooser`)

`ChooseApplication(handle, app_id, parent_window, choices, options) ->
(response, results)`: `choices` are desktop ids without `.desktop`;
options `last_choice` (`s`), `modal` (`b`), `content_type` (`s`), `uri`,
`filename` (`s`), `activation_token` (`s`); results `choice` (`s`) and
`activation_token` (`s`). `UpdateChoices(handle, choices)` replaces the
list while the dialog is open.

### Helix

A dialog "Open With" with a list of the choices by name (apps without an
installed desktop file are left out), the `last_choice` selected at first
(else the first), message "Choose an app to open <file name>." (from
`uri` or `filename`, else "Choose an app."), buttons "Open" and "Cancel".
Response 0 with `choice` the selected id; 1 when canceled or when no
choice is left. `UpdateChoices` for a request whose dialog is open
replaces the list (the shell's `UpdateApps`, `org.helix.PortalDialog.xml`;
installed apps only, the selection staying on the same app when it is
still listed, else the first); with no installed app left the dialog is
canceled (response 1). For any other handle it fails with
`org.freedesktop.DBus.Error.InvalidArgs`.

## Background (`org.freedesktop.impl.portal.Background`)

- `GetAppState() -> apps`: `a{sv}`, app id to `u` state: 0 background
  (running without windows), 1 running, 2 active (focused).
- `NotifyBackground(handle, app_id, name) -> (response, results)`: an app
  keeps running with no windows; results `result` (`u`): 0 forbid, 1
  allow, 2 allow once.
- `EnableAutostart(app_id, enable, commandline, flags) -> result`: start
  the app at login with `commandline`; flags 1: D-Bus activatable.
- Signal `RunningApplicationsChanged`: the state changed.

### Helix

- States come from the shell's windows (`org.helix.PortalDialog.RunningApps`):
  apps with windows are 1, the focused one 2. Apps without windows are not
  listed (the shell does not know them; see `gaps.md`).
  `RunningApplicationsChanged` follows the shell's `RunningAppsChanged`.
- `NotifyBackground`: a dialog "Background Activity" with "<name> is
  running in the background.", buttons "Allow" and "Forbid": result 1 or
  0.
- `EnableAutostart`: writes (or with `enable` false removes)
  `$XDG_CONFIG_HOME/autostart/<app_id>.desktop`: `Type=Application`,
  `Name=<app_id>`, `Exec=` the command line (each argument quoted as the
  Desktop Entry specification says), `X-Flatpak=<app_id>`, and
  `DBusActivatable=true` with flag 1; result true when that worked.

## Wallpaper (`org.freedesktop.impl.portal.Wallpaper`)

- `SetWallpaperURI(handle, app_id, parent_window, uri, options) ->
  response`: set the picture at `uri` (the frontend hands local files over
  as `file:` URIs). Options: `show-preview` (`b`): let the user see the
  picture first; `set-on` (`s`): `background`, `lockscreen` or `both`
  (the default). No results.

### Helix

- Only local files are accepted; anything else, or an unknown `set-on`,
  answers 2.
- With `show-preview`, the shell's dialog (`org.helix.PortalDialog.Dialog`)
  shows "Set Background", "<app name> wants to set this picture as the
  background / the lock screen / the background and lock screen.", the
  picture scaled to fit, and buttons "Cancel" (answer 1) and "Set".
- The file is copied to `$XDG_DATA_HOME/backgrounds/` (the app's copy may
  go away), named `<YYYY-MM-DD-HH-MM-SS>-<original name>`, and the copy's
  URI is written to `org.gnome.desktop.background` `picture-uri` and
  `picture-uri-dark`, and/or `org.gnome.desktop.screensaver`
  `picture-uri`. A `picture-options` of `none` becomes `zoom` so the
  picture shows.

## Email (`org.freedesktop.impl.portal.Email`)

- `ComposeEmail(handle, app_id, parent_window, options) -> (response,
  results)`: open a new message for the user to finish and send. Options:
  `address` (`s`) and `addresses` (`as`): recipients; `cc`, `bcc` (`as`);
  `subject`, `body` (`s`); `attachments` (`as`): the files to attach
  (the frontend passes local files); `activation_token` (`s`): for the mail
  app to raise its window. No results. The interface has no properties.

### Helix

- No dialog: the user's default handler of `mailto:` links (the
  `x-scheme-handler/mailto` default application) opens with the message.
  Without one the answer is 2.
- Recipients that are not of the form `user@host`, or that contain
  whitespace, control characters, `?`, `&`, `,`, quotes or angle
  brackets, or start with `-`, are left out. Attachments may be paths or
  `file:` URIs.
- The handler gets a `mailto:` URI (RFC 6068): the recipients, then `cc`,
  `bcc`, `subject`, `body` and one `attach` field per attachment (a path),
  values percent-encoded except letters, digits, `-._~@`. When the
  handler's program is named `thunderbird…` (run directly, not through a
  sandbox launcher), it gets instead `-compose` and Thunderbird's
  `to='…',cc='…',bcc='…',subject='…',body='…',attachment='<file URIs>'`
  list (single quotes are dropped from values), since it does not attach
  files named in a `mailto:` URI.
- `activation_token` is passed to the mail app as `XDG_ACTIVATION_TOKEN`
  and `DESKTOP_STARTUP_ID`.

## DynamicLauncher (`org.freedesktop.impl.portal.DynamicLauncher`)

Apps (web browsers above all) add launchers to the user's apps. The
frontend writes the desktop file and icon itself; the backend only lets
the user confirm.

- `PrepareInstall(handle, app_id, parent_window, name, icon_v, options)
  -> (response, results)`: show the launcher (`icon_v` is a serialized
  `GBytesIcon`: `('bytes', <ay>)`, PNG, JPEG or SVG data the frontend has
  checked) with its default `name`. Options: `modal` (`b`, default true);
  `launcher_type` (`u`, default 1); `target` (`s`): a web app's address,
  to show; `editable_name` (`b`, default true); `editable_icon` (`b`,
  default false). Results: `name` (`s`), the name to use, and `icon`
  (`v`), the icon to use.
- `RequestInstallToken(app_id, options) -> response`: 0 lets the app add
  launchers without the dialog (with a token the frontend hands out);
  anything else refuses.
- Properties: `SupportedLauncherTypes` (`u`, bits: 1 application, 2 web
  app) and `version` (`u`).

### Helix

- `SupportedLauncherTypes` 3, `version` 1. Another `launcher_type` answers 2.
- The shell's dialog (`org.helix.PortalDialog.Dialog`): "Add to Apps",
  "<app name> wants to add a launcher for this app / web app to your
  apps." (with a name the user cannot change: "… for the app / web app
  “<name>” to your apps."), a web app's `target` below it, the icon, a
  "Name" field holding the name when it is editable, and buttons "Cancel"
  (answer 1) and "Add".
- The name is trimmed, its runs of whitespace (newlines too) made single
  spaces and cut to 255 characters; an empty name means the default one.
  With `editable_icon` the dialog also has a check box "Choose a
  different icon" (off); accepting with it on opens Helix's file chooser
  ("Choose an Icon", button "Choose", in the Pictures folder, else home,
  showing PNG, JPEG and SVG images). The chosen file becomes the icon
  (`('bytes', <ay>)`) when it is at most 4 MiB and starts like a PNG or
  JPEG image or holds `<svg` in its first 4 KiB; otherwise, or when the
  chooser is canceled, the app's icon is returned unchanged (the name
  still counts: the launcher is added).
  `modal` has no effect (Helix's dialogs are always modal).
- `RequestInstallToken` answers 2 for every app: Helix lets no app add
  launchers without asking. (Which apps GNOME's backend trusts is not
  documented here.)

## InputCapture (`org.freedesktop.impl.portal.InputCapture`)

Apps that share a keyboard and mouse with other computers (Input Leap,
Synergy) capture the local input once the pointer leaves the screen
through a barrier they set, and get it as an EIS (libei) receiver.

- `CreateSession(handle, session_handle, app_id, parent_window, options)
  -> (response, results)`: `capabilities` (`u`: 1 keyboard, 2 pointer, 4
  touchscreen) asked; results `capabilities`, those granted.
- `GetZones(handle, session_handle, app_id, options)`: results `zones`
  (`a(uuii)`: each zone's width, height, x and y, in logical
  coordinates) and `zone_set` (`u`), naming this set of zones.
- `SetPointerBarriers(handle, session_handle, app_id, options, barriers,
  zone_set)`: each barrier a vardict with `barrier_id` (`u`, non-zero) and
  `position` (`(iiii)`: x1, y1, x2, y2; horizontal or vertical). Results:
  `failed_barriers` (`au`).
- `Enable`, `Disable`, `Release(session_handle, app_id, options)`:
  `Release` takes `activation_id` (`u`) and optionally `cursor_position`
  (`(dd)`), where the pointer goes.
- `ConnectToEIS(session_handle, app_id, options) -> fd`.
- Signals (each with the session handle and a vardict): `Activated`
  (`activation_id`, `barrier_id`, `cursor_position`) when input is
  captured; `Deactivated` (`activation_id`, `cursor_position`) when it is
  given back; `Disabled` when the session can no longer capture until
  enabled again; `ZonesChanged` (`zone_set`).
- Properties `SupportedCapabilities` (`u`) and `version` (`u`). Version 2
  adds `CreateSession2` and `Start` (creating, then asking), with
  `persist_mode`/`restore_data` and a clipboard.

### Helix

- `version` 2, `SupportedCapabilities` 3 (keyboard and pointer;
  touchscreens are not captured). `CreateSession2` makes the session
  (nothing is asked); `Start` asks and starts it. `CreateSession` (version
  1) does both.
- With `persist_mode` 1 or 2, `Start`'s results carry `restore_data`
  `("Helix", 1, <a{sv}>)` holding `devices` (`u`), the capabilities
  granted. A `Start` whose `restore_data` is Helix's and grants every
  capability asked for starts without asking. A session that asked for
  the clipboard (`RequestClipboard`, see "Clipboard") shares it once
  started: no separate question; `clipboard_enabled` says whether it
  does.
- Starting asks with the shell's dialog (`org.helix.PortalDialog`):
  "Allow Input Capture?", "<app name> wants to take over the pointer and
  keyboard when the pointer leaves the screen. <key> gives them
  back." (`<key>`: the first `org.gnome.mutter.keybindings`
  `cancel-input-capture` accelerator written as `Super+Shift+Escape`), "Deny" (answer 1) and "Allow". Allowed, the compositor makes the
  session (`org.helix.InputCapture.xml`); asking for no capability Helix
  has answers 2.
- The zones are the monitors' logical areas; the zone set changes when
  they do (and the session's barriers are then dropped, with
  `ZonesChanged`). `SetPointerBarriers` with an old zone set refuses every
  barrier; otherwise a barrier is refused when its id is 0, it is
  diagonal, or it does not lie on an edge of a zone (on a side's line,
  within it).
- An enabled session with an EIS client captures when the pointer moves
  across one of its barriers (the move's start and end on two sides of
  the barrier's line, crossing it within the barrier); the pointer stays
  where the move stopped, which `Activated` reports. Activation ids count
  up from 1. While captured the pointer is hidden and every key, button,
  scroll and pointer motion (from devices and remote desktop sessions)
  goes to the session's EIS clients instead of the desktop; absolute
  pointer positions and touch are not captured.
- The EIS client must be a receiver. Its seat offers a keyboard (with the
  current keymap) and a pointer (motion, buttons, scrolling), as the
  session's capabilities allow; while captured the devices are emulating,
  with the activation id as the sequence number. Events come one per
  frame.
- `Release` with the current activation id ends the capture (the pointer
  goes to `cursor_position`, kept on the monitors) and emits
  `Deactivated`; `Disable` ends a capture too. `cancel-input-capture`
  (default Super+Shift+Escape; its modifiers are taken from the keys
  held, its key from the layout's symbol without modifiers) ends a
  capture at any time: the client gets the release of the other keys it
  saw held, the session is
  disabled, and `Deactivated` then `Disabled` are emitted. A capture also
  ends when its EIS client disconnects.
- Closing the portal session closes the compositor's, and the other way
  round.

## GlobalShortcuts (`org.freedesktop.impl.portal.GlobalShortcuts`)

An app (through the frontend) makes a session, binds shortcuts to it and
is told when they are pressed and released, whatever window has focus.

- `CreateSession(handle, session_handle, app_id, options)`: a session
  object (`org.freedesktop.impl.portal.Session`) at `session_handle`.
- `BindShortcuts(handle, session_handle, shortcuts, parent_window,
  options)`: `shortcuts` are `(id, properties)` with `description` (`s`)
  and optionally `preferred_trigger` (`s`, in the XDG shortcuts format:
  modifiers `CTRL`, `ALT`, `SHIFT`, `LOGO`, `NUM` and a key name, joined
  by `+`, e.g. `CTRL+SHIFT+a`). Results: `shortcuts` (`a(sa{sv})`), each
  with `description` and `trigger_description` (`s`, human-readable).
- `ListShortcuts(handle, session_handle)`: results `shortcuts`, as bound.
- Signals `Activated` / `Deactivated(session_handle, shortcut_id,
  timestamp, options)` when a bound shortcut is pressed / released;
  `ShortcutsChanged(session_handle, shortcuts)` when the bindings change.
- Closing the session releases its shortcuts.

### Helix

- Version 1 (no `ConfigureShortcuts`; triggers cannot be changed).
- `BindShortcuts` asks the user, unless the app's shortcuts were allowed
  before with the same triggers: a dialog "Allow Global Shortcuts?"
  "<app> wants to use these shortcuts in every app.", one row per
  shortcut ("<description> — <trigger>", or "— not assigned"), **Allow**
  / **Deny** (denying answers 1, cancelled). An allowed set is remembered
  for the app in `$XDG_DATA_HOME/helix/global-shortcuts` (one line per
  shortcut: app id, shortcut id, trigger, tab-separated).
- A shortcut is bound when its `preferred_trigger` converts to an
  accelerator: `CTRL` → `<Control>`, `ALT` → `<Alt>`, `SHIFT` →
  `<Shift>`, `LOGO` → `<Super>`, `NUM` is dropped; the key name as given.
  Shortcuts without one, or whose accelerator the shell will not grab
  (taken, or invalid), are listed with `trigger_description` "Not
  assigned" and never fire.
- Binding goes through the shell's `org.gnome.Shell`
  `GrabAccelerators`, in the normal and overview action modes, with
  trigger-on-release; `AcceleratorActivated` / `AcceleratorDeactivated`
  become `Activated` / `Deactivated` with the shell's `timestamp`
  (milliseconds) and empty options. Binding again replaces the session's
  shortcuts (the old ones are released, then `ShortcutsChanged`).
- `trigger_description` writes modifiers as `Ctrl`, `Alt`, `Shift`,
  `Super` and the key with a capital first letter, joined by `+` (e.g.
  `Ctrl+Shift+S`).

## Lockdown (`org.freedesktop.impl.portal.Lockdown`)

Read-write boolean properties apps check before offering a feature:
`disable-printing`, `disable-save-to-disk`, `disable-application-handlers`,
`disable-location`, `disable-camera`, `disable-microphone`,
`disable-sound-output`.

### Helix

Each property is a GSettings key: the first three are the same-named keys
of `org.gnome.desktop.lockdown`, `disable-location` is the opposite of
`org.gnome.system.location` `enabled`, and the last three are the
same-named keys of `org.gnome.desktop.privacy` (key tables in
`contracts/gsettings/`). Setting a property writes the key (an error when
the key is not writable). `PropertiesChanged` is emitted when a key
changes.

## Notification (`org.freedesktop.impl.portal.Notification`)

- `AddNotification(app_id, id, notification)`: show (or replace) app
  `app_id`'s notification `id`. `notification` holds the keys GLib's
  `GNotification` uses (`title`, `body`, `icon`, `priority`,
  `default-action`, `default-action-target`, `buttons`; see
  `org.gtk.Notifications.md`); version 2 of the interface adds more
  (`markup-body`, `sound`, `category`, `display-hint`, icons and sounds as
  file descriptors).
- `RemoveNotification(app_id, id)`: withdraw it.
- Signal `ActionInvoked(app_id, id, action, parameter)`: the user chose
  `action` (with its target in `parameter`, if any); the frontend passes
  it to the app.
- Property `version`.

### Helix

- `version` is 2. Property `SupportedOptions` (`a{sv}`): `category`,
  the categories Helix takes: `im.received`, `alarm.ringing`,
  `call.incoming`, `call.ongoing`, `call.unanswered`,
  `weather.warning.extreme`, `cellbroadcast.danger.extreme`,
  `cellbroadcast.danger.severe`, `cellbroadcast.amber-alert`,
  `cellbroadcast.test`, `os.battery.low`, `browser.web-notification`; and
  `button-purpose`: `im.reply-with-text`, `call.accept`, `call.decline`,
  `call.hang-up`, `call.enable-speakerphone`,
  `call.disable-speakerphone` (buttons are passed on with their
  `purpose`; see `org.gtk.Notifications.md` for what the shell does).
- Keys taken: `title`, `body`, `markup-body` (shown with notification
  markup in place of `body`), `icon` (themed, bytes, or — version 2 —
  `("file-descriptor", h)`, whose contents are read and passed on as
  bytes), `priority`, `default-action`, `default-action-target`,
  `buttons`, and:
  - `category`: `alarm.ringing`, `call.incoming`,
    `weather.warning.extreme`, `cellbroadcast.danger.extreme`,
    `cellbroadcast.amber-alert` and `os.battery.low` make the
    notification urgent (unless it has its own `priority`); the others
    change nothing;
  - `sound`: `default` plays the sound theme's `message-new-instant`;
    `silent` or no `sound`, nothing; `("file-descriptor", h)` plays that
    file (copied to `$XDG_RUNTIME_DIR/helix-notification-sounds/`);
  - `display-hint`: `transient` makes the banner go away without staying
    in the notification list; the other hints are ignored.
  Other keys are dropped.
- The notification is passed to the shell's `org.gtk.Notifications`
  `AddNotification` (and `RemoveNotification`) with the keys above
  (sounds and transience as the Helix-private keys `x-helix-sound`, `s`:
  a sound name or an absolute file, and `x-helix-transient`, `b`), so it looks and
  behaves as a GTK notification of app `app_id`, with that app's
  notification settings.
- Actions: the shell activates `app.` actions over
  `org.freedesktop.Application` at the app's bus name; when the app cannot
  be reached there (no owner and not activatable, as for many portal
  clients), the shell sends `org.gtk.Notifications.ActionInvoked` instead
  (`org.gtk.Notifications.md`, "Activation"). The backend passes those for
  the notifications it added on as this interface's `ActionInvoked`, with
  the same action name; the parameter is the target (if any), then the
  platform data (`a{sv}`, the shell's, without its Helix-private keys),
  then the user's reply (`s`) for an `im.reply-with-text` button. The
  shell gives the reply as the last parameter value and, as
  `x-helix-response`, in its platform data; the backend moves it.

## Print (`org.freedesktop.impl.portal.Print`)

- `PreparePrint(handle, app_id, parent_window, title, settings,
  page_setup, options) -> (response, results)`: let the user choose how
  to print. `settings` and `page_setup` are GTK's print settings and page
  setup as string-keyed dictionaries (`GtkPrintSettings`, `GtkPageSetup`
  serializations: `printer`, `n-copies`, `paper-format`, `PPDName`, …);
  options `modal` (`b`), `accept_label` (`s`). Results: `settings`,
  `page-setup`, and `token` (`u`) for a later `Print`.
- `Print(handle, app_id, parent_window, title, fd, options) -> (response,
  results)`: print the document in `fd` (PDF, or PostScript). Options:
  `token` (`u`) from `PreparePrint` (then no dialog is needed), `modal`.

### Helix

- With `org.gnome.desktop.lockdown` `disable-printing` true, both answer
  2 without a dialog.
- The dialog (`org.helix.PortalDialog.Dialog`): "Print", "Print “<title>”
  on:" (or "Choose a printer."), a list of the CUPS printers (`lpstat
  -e`; the default one, `lpstat -d`, first and chosen; for `PreparePrint`
  the `printer` of `settings` when listed) and "Print to File", a "Copies"
  field (1 to 999; others are refused with a warning), buttons "Cancel"
  and `accept_label` (default "Print").
- `PreparePrint` answers `settings` with `printer` set to the chosen
  printer (or "Print to File") and `n-copies` to the copies, `page-setup`
  as given, and a new `token`; the choice is kept for that token (until
  used). `Print` with a known token uses its choice; otherwise it shows
  the dialog first.
- A printer: the document goes to `lp -d <printer> -n <copies> -t
  <title> -` on its standard input; response 0 when `lp` succeeds.
- Print to File: the document is saved in the user's Documents folder
  (else the home folder) as `<title>.pdf` (`.ps` for PostScript; `/` in
  the title becomes `-`; "output" without a title), with " (2)" etc. when
  the name is taken. Copies do not apply.
- Paper size, page setup, ranges and other print settings are passed
  through to the app, not shown.

## FileChooser (`org.freedesktop.impl.portal.FileChooser`)

- `OpenFile(handle, app_id, parent_window, title, options) -> (response,
  results)`. Options: `accept_label` (`s`), `modal` (`b`), `multiple`
  (`b`), `directory` (`b`: choose folders), `filters` (`a(sa(us))`: each a
  name and patterns, kind 0 a glob such as `*.png`, kind 1 a MIME type),
  `current_filter` (`(sa(us))`), `choices` (`a(ssa(ss)s)`: id, label,
  options (id, label), initial value; no options means a check box with
  `true`/`false`), `current_folder` (`ay`, a NUL-terminated path).
  Results: `uris` (`as`), `choices` (`a(ss)`), `current_filter`.
- `SaveFile(...)`: options as above (no `multiple`, `directory`) plus
  `current_name` (`s`) and `current_file` (`ay`). Results: `uris` (one),
  `choices`, `current_filter`.
- `SaveFiles(...)`: options `accept_label`, `modal`, `choices`,
  `current_folder`, `files` (`aay`: file names). Results: `uris`, one per
  file, in the chosen folder.

### Helix

The shell's dialog (`org.helix.PortalDialog.Dialog`) shows one folder at a
time: the title (or the accept label), the folder as the message (`~` for
the home folder), a list of rows, a text field, the check box of the first
choice without options (the others keep their initial values), and Cancel
and accept buttons (`accept_label`, else "Open", "Select" for folders,
"Save"). Clicking the current row again acts as accepting it.

- The rows, in order: "Select This Folder" (folders) or "Save Here"
  (saving); "Parent Folder" (except at `/`); "File type: <name> (choose to
  change)" when there are two or more filters (not for folders or
  `SaveFiles`); the folders (`<name>/`), then the files (not when choosing
  folders or for `SaveFiles`), each sorted by name ignoring case. Names
  starting with `.` are hidden; files are shown when the current filter
  matches them (globs ignore case; MIME types by the type guessed from the
  name).
- Left of the rows, a column of places: "Recent" (only when opening
  files), "Home", the user's standard folders (Desktop, Documents,
  Downloads, Music, Pictures, Videos, as `user-dirs.dirs` names them,
  except the home folder itself), GTK bookmarks
  (`$XDG_CONFIG_HOME/gtk-3.0/bookmarks`: local `file:` URIs, named by
  their label or the folder's name), drives mounted under
  `/media/<user>/` and `/run/media/<user>/` (by folder name), network
  places the user connected to (GVfs's FUSE folders under
  `$XDG_RUNTIME_DIR/gvfs/`, named "<share> on <server>" for
  `smb-share:server=…,share=…`, "<user> on <host>" or "<host>" for others,
  else by folder name; checked with unit tests only), and "Computer"
  (`/`); folders that do not exist are left out, and a folder
  is listed once. The place showing is marked. Clicking a place shows it
  (when saving, the name typed so far is kept). Recent lists recently
  used files (`$XDG_DATA_HOME/recently-used.xbel`), newest first by when
  they were last modified there (else added), at most 100, only local
  files that still exist and pass the current filter, each as "<name> —
  <folder name>"; the message then reads "Recent". It lists nothing
  while `org.gnome.desktop.privacy remember-recent-files` is false or
  `recent-files-max-age` is 0, and with a positive age only files used
  within that many days. Recent files are chosen
  (and, with `multiple`, picked) as files are. The file type row is
  shown there too; there is no Parent Folder.
- Previews: when some rows can be previewed (files, or recent files,
  that are pictures — PNG, JPEG, WebP, JPEG XL or SVG —, text, or of a
  type an installed thumbnailer handles; by content type, from the name
  and the first 4 KiB), an area below the rows shows the selected row's
  preview, or "No preview":
  - a picture scaled down to fit 392 × 160 pixels (not above 32 MiB);
  - text: its first nine lines (of the first 8 KiB), tabs as four
    spaces;
  - other files: their thumbnail, from the freedesktop thumbnail cache
    (`$XDG_CACHE_HOME/thumbnails/{x-large,large,normal}/<md5 of the
    URI>.png`, used when not older than the file), else Helix's own cache
    (`$XDG_CACHE_HOME/helix/thumbnails/`, same names), else made into
    Helix's cache by the thumbnailer whose `.thumbnailer` file (in
    `thumbnailers/` under the XDG data folders; skipped when its `TryExec`
    is missing) lists the type, run with `%i` / `%u` (the file's path /
    URI), `%o` (the output) and `%s` (256), and stopped after 10 seconds.
    Helix does not write into the shared cache (its entries need metadata
    Helix's thumbnails lack).

  The shell makes each preview once per folder view, without holding up
  the dialog.
- The start folder: `current_folder`, else `current_file`'s folder, else
  the folder the app last chose in (this session), else the home folder.
  The first filter is current unless `current_filter` names one (added
  when it is not among `filters`).
- Accepting a row: Parent Folder goes up (with the folder left as the
  current row); a folder opens; the file type row steps to the next
  filter; Select This Folder chooses the folder shown; a file is chosen
  (when saving: its name is put in the Name field).
- Opening: the text field is "Location": a path typed there (relative to
  the folder shown, or absolute, or starting with `~`) opens a folder or
  chooses a file; otherwise, when opening files and the text has no
  `/`, it searches the folder shown and the folders below it (see
  "Searching files" below). The results are
  rows "<name> — <folder name>" (chosen as files are) under the message
  "Search for “<text>” in <folder>"; with none, the warning "No files
  named like “<text>”.". Clicking a place ends the search. Otherwise the
  warning "“<text>” was not found.". With
  `multiple`, files are picked by clicking them (or Space) and accepting
  chooses all picked files; folders and the other rows are not picked.
- Saving: the text field is "Name" (`current_name`, else `current_file`'s
  name). Save Here saves under that name in the folder shown (a name with
  `/` is a path as above; a folder name opens it); an empty name gives the
  warning "Type a name for the file.", a missing folder "That folder does
  not exist.". An existing file needs confirming: "Replace File?", "A file
  named “<name>” already exists in “<folder>”. Replacing it overwrites
  its contents.", buttons "Cancel" (back to the chooser) and "Replace".
  `SaveFiles`: Save Here answers each of `files`' names in the folder
  shown (existing files are not asked about).
- With `org.gnome.desktop.lockdown` `disable-save-to-disk` true, `SaveFile`
  and `SaveFiles` answer 2 without a dialog.
- Results: `uris` as `file:` URIs; `choices` with every choice's value
  (the check box's as `true`/`false`); `current_filter` when there are
  filters.

### Searching files

When a file index runs on the session bus (localsearch,
`org.freedesktop.LocalSearch3`, else Tracker 3's
`org.freedesktop.Tracker3.Miner.Files`; neither is started for a
search), the chooser asks it first, through its SPARQL endpoint
(`org.freedesktop.Tracker3.Endpoint` at `/org/freedesktop/Tracker3/Endpoint`,
`Query(s sparql, h fd, a{sv} options) → as`: the column names; the
results are written to `fd`). Helix's query asks for the `nie:url` of
files (up to 1000) below the folder whose file name (`nfo:fileName`) or
whose content (an information element with `nie:isStoredAs` the file)
`fts:match`es the text's words, each as a word prefix (`budg` finds
"budget"), all of them required; characters other than letters and digits
separate words. As observed with Tracker 3.7, the results written to `fd`
are, per row: the number of columns, each column's value type, each
value's end offset counted from the first value's start, then the
values, each ending in a NUL byte; the numbers are 32-bit integers in
the machine's byte order. Of the files found, those still there, not in
a hidden folder below the folder shown and passing the current filter
are shown, those whose names hold the text first, then by depth and
name; at most 200.

With no index running, or when it finds none of them, the chooser
scans: the folder shown and the folders below it (hidden ones skipped)
for files whose names hold the text (ignoring case) and pass the current
filter, nearest folders first, then by name; after them, text files (by
their content type; at most 256 KiB each, the first 500 such files met)
whose content holds the text (ignoring case); at most 200 results, after
looking at 20 000 entries at most.
