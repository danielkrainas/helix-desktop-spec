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
  `cursor_mode` is not 1, 2 or 4 (closing the session). `multiple` is
  accepted, but one source is shared.
- When `types` is only virtual (4), `Start` shares a new virtual monitor
  (`RecordVirtual`) without asking: nothing that exists is shown. Its
  stream has `position`, `size` and `source_type` 4. Otherwise virtual
  is not offered.
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
another portal (RemoteDesktop) asks for the clipboard with
`RequestClipboard(session_handle, options)` before it starts; once
started, `SetSelection`, `SelectionWrite`, `SelectionWriteDone` and
`SelectionRead` take the session handle, and `SelectionOwnerChanged`
(`mime_types` `as`, `session_is_owner` `b`) and `SelectionTransfer` are
emitted with it. `version` is 1.

### Helix

- Only RemoteDesktop sessions: `RequestClipboard` for any other or
  unknown session fails with `org.freedesktop.DBus.Error.InvalidArgs`;
  after `Start` with `org.freedesktop.DBus.Error.Failed`.
- No separate question: allowing the session (the RemoteDesktop `Start`
  dialog) allows the clipboard it asked for. `Start` then turns on the
  compositor session's clipboard (`EnableClipboard`, see
  `org.gnome.Mutter.RemoteDesktop.md`) and returns `clipboard_enabled`
  true (false when not asked for, or when that fails).
- The methods and signals are those of the compositor session, passed
  through (option names with `_` instead of `-`); for a session that does
  not share the clipboard the methods fail with
  `org.freedesktop.DBus.Error.Failed`.

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
choice is left. `UpdateChoices` is accepted and ignored (see `gaps.md`).

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
  The icon is always returned unchanged (`editable_icon` is not offered).
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
  capability asked for starts without asking. `clipboard_enabled` is
  always false: Helix does not share the clipboard with input capture
  sessions.
- Starting asks with the shell's dialog (`org.helix.PortalDialog`):
  "Allow Input Capture?", "<app name> wants to take over the pointer and
  keyboard when the pointer leaves the screen. Super+Escape gives them
  back.", "Deny" (answer 1) and "Allow". Allowed, the compositor makes the
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
  `Deactivated`; `Disable` ends a capture too. Super+Escape ends a capture
  at any time: the client gets the Super key's release, the session is
  disabled, and `Deactivated` then `Disabled` are emitted. A capture also
  ends when its EIS client disconnects.
- Closing the portal session closes the compositor's, and the other way
  round.

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

- `version` is 1, so the frontend sends version 1 notifications; Helix
  also takes `markup-body` (shown with notification markup in place of
  `body`). Other keys are dropped.
- The notification is passed to the shell's `org.gtk.Notifications`
  `AddNotification` (and `RemoveNotification`) unchanged, so it looks and
  behaves as a GTK notification of app `app_id`, with that app's
  notification settings.
- Actions: the shell activates `app.` actions over
  `org.freedesktop.Application` at the app's bus name; when the app cannot
  be reached there (no owner and not activatable, as for many portal
  clients), the shell sends `org.gtk.Notifications.ActionInvoked` instead
  (`org.gtk.Notifications.md`, "Activation"). The backend passes those for
  the notifications it added on as this interface's `ActionInvoked`, with
  the same action name and parameter.

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
  <folder name>"; the message then reads "Recent". Recent files are chosen
  (and, with `multiple`, picked) as files are. The file type row is
  shown there too; there is no Parent Folder.
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
  `/`, it searches: the folder shown and the folders below it (hidden
  ones skipped) for files whose names hold the text (ignoring case) and
  pass the current filter, nearest folders first, then by name; at most
  200 results, after looking at 20 000 entries at most. The results are
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
