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

`helix-portals.conf` (used while `XDG_CURRENT_DESKTOP` contains `Helix`)
has `default=helix;gtk;`: interfaces Helix does not serve go to the GTK
backend when it is installed. GNOME's backend is not listed: it expects
GNOME Shell, and since the frontend asks every listed backend for
settings, a GNOME backend that could not start stalled apps for its
25-second activation timeout.

*GNOME does not document this; Helix's choice:* GNOME's backends do more
than the documented interfaces (their dialogs and wording, how the mail
app and its arguments are chosen, which apps get DynamicLauncher install
tokens, what is done with the lockdown keys beyond the portal); Helix's
behaviour is in each "Helix" section below.

## Request (`org.freedesktop.impl.portal.Request`)

Methods that take a `handle` export a Request object at that path for the
duration of the interaction. The frontend calls `Close` to end it, after
which the object must be removed.

## Session (`org.freedesktop.impl.portal.Session`)

Methods that take a `session_handle` export a Session object at that path.
`Close` ends the session; `Closed` is emitted when the backend ends it.

*GNOME does not document this; Helix's choice:* Helix's Session objects
report `version` 1.

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
are undocumented, yet sandboxed GTK reads `org.gnome.desktop.interface`
(and others) through the portal.

*GNOME does not document this; Helix's choice:* which namespaces and keys
are exposed, how `accent-color` names map to sRGB (the colours libadwaita
documents), and the `version` property (1), as below.

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

`version` is 1.

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

*GNOME does not document this; Helix's choice:* GNOME's `restore_data`
contents are its own; Helix's are below.

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
  (`org.helixdesktop.ScreenCastChooser.md`), offering the types asked for, and
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
  selected, the share dialog (`org.helixdesktop.ScreenCastChooser.md`) also says
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

*GNOME does not document this; Helix's choice:* where non-interactive
screenshots are saved and when the user is asked, as below; the window
pick is Helix's own design.

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
  compositor session is its `org.helixdesktop.InputCapture` session, which
  shares the clipboard the same way (`org.helixdesktop.InputCapture.xml`); its
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

The shell's dialog (`org.helixdesktop.PortalDialog.Ask`): title the `title`,
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
replaces the list (the shell's `UpdateApps`, `org.helixdesktop.PortalDialog.xml`;
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

- States come from the shell's windows (`org.helixdesktop.PortalDialog.RunningApps`):
  apps with windows are 1, the focused one 2. Apps without windows are not
  listed (the shell does not know them).
  `RunningApplicationsChanged` follows the shell's `RunningAppsChanged`.
- `NotifyBackground`: a dialog "Background Activity" with "<name> is
  running in the background.", buttons "Allow" and "Forbid": result 1 or
  0.
- `EnableAutostart`: writes (or with `enable` false removes)
  `$XDG_CONFIG_HOME/autostart/<app_id>.desktop`: `Type=Application`,
  `Name=<app_id>`, `Exec=` the command line (each argument quoted as the
  Desktop Entry specification says, then backslashes doubled as key-file
  strings want), `X-Flatpak=<app_id>`, and
  `DBusActivatable=true` with flag 1; result true when that worked. The
  file is written whole (a new file renamed into place). Nothing is
  written, and the result is false, for an `app_id` other than letters,
  digits, `.`, `_` and `-`, an empty command line, or an argument with a
  control character (a newline would start a key of its own).

## Wallpaper (`org.freedesktop.impl.portal.Wallpaper`)

- `SetWallpaperURI(handle, app_id, parent_window, uri, options) ->
  response`: set the picture at `uri` (the frontend hands local files over
  as `file:` URIs). Options: `show-preview` (`b`): let the user see the
  picture first; `set-on` (`s`): `background`, `lockscreen` or `both`
  (the default). No results.

### Helix

- Only local files are accepted; anything else, or an unknown `set-on`,
  answers 2.
- With `show-preview`, the shell's dialog (`org.helixdesktop.PortalDialog.Dialog`)
  shows "Set Background", "<app name> wants to set this picture as the
  background / the lock screen / the background and lock screen.", the
  picture scaled to fit, and buttons "Cancel" (answer 1) and "Set".
- The file is copied to `$XDG_DATA_HOME/backgrounds/` (the app's copy may
  go away), named `<YYYY-MM-DD-HH-MM-SS>-<original name>`, and the copy's
  URI is written to `org.gnome.desktop.background` `picture-uri` and
  `picture-uri-dark`, and/or `org.gnome.desktop.screensaver`
  `picture-uri`. A `picture-options` of `none` becomes `zoom` so the
  picture shows. Agent Access's `SetWallpaper` copies the same way
  (`contracts/agent-access/README.md`, "Changing settings").

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
- The shell's dialog (`org.helixdesktop.PortalDialog.Dialog`): "Add to Apps",
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

*GNOME does not document this; Helix's choice:* mutter's input capture
interface (`org.gnome.Mutter.InputCapture`) is undocumented, so the
compositor's side is Helix's private `org.helixdesktop.InputCapture`; how
barriers, keys held at activation and escaping a capture work is as below.

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
- Starting asks with the shell's dialog (`org.helixdesktop.PortalDialog`):
  "Allow Input Capture?", "<app name> wants to take over the pointer and
  keyboard when the pointer leaves the screen. <key> gives them
  back." (`<key>`: the first `org.gnome.mutter.keybindings`
  `cancel-input-capture` accelerator written as `Super+Shift+Escape`), "Deny" (answer 1) and "Allow". Allowed, the compositor makes the
  session (`org.helixdesktop.InputCapture.xml`); asking for no capability Helix
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

*GNOME does not document this; Helix's choice:* how the user is asked,
where answers are kept, and the unit of the signals' `timestamp`
(milliseconds), as below.

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
  (milliseconds) and empty options. Only those signals sent by the
  current owner of `org.gnome.Shell` count (the owner is followed through
  `NameOwnerChanged`); the same signals from any other client are
  ignored, so no other session-bus client can fire an app's shortcuts.
  Binding again replaces the session's shortcuts (the old ones are
  released, then `ShortcutsChanged`).
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

*GNOME does not document this; Helix's choice:* how categories and
display hints are shown, which sound `default` means, and which button
purposes are offered, as below.

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
- The dialog (`org.helixdesktop.PortalDialog.Dialog`): "Print", "Print “<title>”
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
  the name is taken. The file is created new (`O_CREAT|O_EXCL`, never
  through a symbolic link): any existing entry of that name, a dangling
  link too, counts as taken. Copies do not apply.
- Documents larger than 512 MiB are not printed (response 2).
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

The backend asks the shell's file chooser (`org.helixdesktop.PortalDialog.ChooseFiles`,
`org.helixdesktop.PortalDialog.xml`) once per request: one dialog stays up for
the whole choice, browsing folders itself, and answers with the chosen
paths. Polkit and keyring prompts wait until it is answered. The backend
turns the paths into `file:` URIs and remembers the folder.

- What is passed: the title (else the accept label), the accept label
  (`accept_label`, else "Open", "Select" for folders, "Save"), what is
  chosen (files, possibly several with `multiple`; a folder with
  `directory`; a name for `SaveFile`; a folder for `SaveFiles`), the
  filters with the current one (`current_filter`, added when it is not
  among `filters`, else the first), the name (`current_name`, else
  `current_file`'s name), the check box of the first choice without
  options (the others keep their initial values), and the start folder:
  `current_folder`, else `current_file`'s folder, else the folder the app
  last chose in (this session). Without one, opening files starts in
  Recent when `org.gtk.gtk4.Settings.FileChooser startup-mode` is
  `recent` (GTK's default), else everything starts in the home folder.
- Size and place: about 80 % of the screen's width (at least 600, at
  most 1180 logical pixels) and 85 % of its height (at least 440, at most
  900), centered, from the screen's size: it neither moves nor changes
  size between folders, and with Large Text the rows grow and fewer show
  while the title, fields and buttons stay on the screen. *GNOME does not
  document this; Helix's choice.*
- Resizing: dragging an edge or a corner of the dialog (6 px either side
  of the edge, 14 px corners; a grip marks the bottom right corner)
  resizes it, kept centered, so an edge moves the opposite one too;
  dragging the gap between the places and the list (a line in the accent
  color while pointed at or dragged) widens or narrows the places. The
  dialog stays between its smallest size from the screen (600 × 440 when
  the screen allows) and the screen less 12 px each side; the places
  between 120 px (or a third of the dialog) and a third of the dialog.
  When the button goes up the size is written to
  `org.gtk.gtk4.Settings.FileChooser` `window-size` and `sidebar-width`,
  which later dialogs (and GTK 4's chooser) start from. The pointer's
  shape does not change over the edges (the shell sets no cursor
  shapes). *Helix's choice.*
- Layout: the title; a row with Back, Forward, the path bar, the list /
  grid switch (`view-grid-symbolic` while the list shows,
  `view-list-symbolic` while the grid does) and, when opening files,
  Search, or when saving or choosing a folder, New Folder; when saving,
  "Name" with the name field; then the places, the
  file list and (while the selected file's type can be previewed) a
  preview pane; at the bottom the check box, a warning line, the file
  type button and Cancel and the accept button (the default).
- Path bar: the home folder (or `/`, "Computer") with its icon, then a
  button per folder down to the one shown, which is bold and pressed;
  going up keeps the folders below as buttons (as GTK's path bar does)
  until another branch is entered. When they do not fit, the first ones
  give way to a "…" button that goes to the deepest folder not shown.
  Clicking a folder's button goes there (with the folder below it
  selected). In Recent the bar reads "Recent", in Starred "Starred", in a
  search "Search for “<text>”", in the Trash "Trash", in Other Locations
  "Other Locations".
- Places (left): Recent (when opening files or saving), Starred
  (`starred-symbolic`; with Recent, while GNOME Files' star store
  exists), Home, the user's
  standard folders (Desktop, Documents, Downloads, Music, Pictures,
  Videos, as `user-dirs.dirs` names them, except the home folder
  itself), the Trash (`user-trash-symbolic`; when opening files only:
  nothing is saved into the Trash, and a folder chosen there would be
  the Trash's own storage), GTK bookmarks (`$XDG_CONFIG_HOME/gtk-3.0/bookmarks`: local
  `file:` URIs, named by their label or the folder's name), drives
  mounted under `/media/<user>/` and `/run/media/<user>/` (by folder
  name), network places the user connected to (GVfs's FUSE folders
  under `$XDG_RUNTIME_DIR/gvfs/`, named "<share> on <server>" for
  `smb-share:server=…,share=…`, "<user> on <host>" or "<host>" for others,
  else by folder name), "Computer"
  (`/`) and "Other Locations" (`list-add-symbolic`), each with a symbolic
  icon, separated into groups (Recent, Starred, standard folders and the Trash,
  bookmarks, drives and network places, Computer and Other Locations);
  folders that do not exist are left out (the Trash is listed whether or
  not its folder exists yet) and a folder is listed once. They are read
  when the dialog opens, again every 2 seconds while it is open, and at
  once when the shell's volume monitor reports a mount coming or going
  (`contracts/gsettings/removable-media.md`), so drives, network places
  and bookmarks appear and go while the dialog is open. When the drive
  or network place holding the folder shown goes, the list goes to the
  home folder. They scroll with the wheel when they do not fit. The
  place shown is marked: the one shown, else the deepest one holding the
  folder shown (a folder inside the Trash marks the Trash). Clicking a
  place shows it.
- Starred: the files and folders starred in GNOME Files (and Helix
  Files) that are inside the home folder and still there, read-only from
  Nautilus's store each time it is shown (`contracts/files/starred.md`),
  with their folder as Location, sorted by the headers like a folder.
  Files go through the file type filter, while folders stay listed so
  they can be opened. Empty: "No starred files". A store that cannot be
  used (no TinySPARQL, a schema Helix does not know) lists as an error.
  The chooser neither stars nor unstars.
- Trash: lists what the freedesktop Trash specification's Trash folders
  hold: the home Trash (`$XDG_DATA_HOME/Trash`) and, for each drive
  among the places, `<drive>/.Trash/<uid>` (when `.Trash` is a sticky
  folder, not a link) and `<drive>/.Trash-<uid>`. An item is listed
  when it has its `info/<name>.trashinfo` and its `files/<name>`, under
  the name it had (the last part of the info's `Path=`, percent-decoded;
  a relative `Path=` is below the drive), with the folder it was in as
  its Location (the Location column, as in Recent, sorted by the
  headers like a folder); its type comes from that name. The items are
  read from the Trash folders themselves (not through GVfs's
  `trash:///`), so choosing one answers the path of the item in
  `files/`, which the app can read, and a trashed folder opens as a
  folder. Empty: "Trash is empty". The chooser neither restores nor
  deletes anything. *GNOME reads the Trash through GVfs; Helix's
  choice, so that the answer is a local path.*
- Other Locations: a list (its Location column the folder, or "Not
  mounted") of "Computer" (`/`), then what the shell's volume monitor
  knows (`contracts/gsettings/removable-media.md`): each mounted drive,
  disc, phone or network share whose root is a local folder (FUSE ones
  too), by its name, and each volume that can be mounted but is not.
  It follows mounts coming and going while it shows. Activating a
  mounted one (double-click, Enter, or the accept button) shows its
  folder; activating one not mounted mounts it with no window opened
  and nothing asked ("Mounting <drive>…" in the warning line; when the
  mount needs a password or an answer, or fails: "The drive could not
  be mounted: <error>"; a volume gone meanwhile: "The drive is gone."),
  after which it shows its folder. Choosing a folder, the accept button
  chooses a mounted one selected. *Helix's choice:* volumes needing a
  password are unlocked from the dock, Files or the notification (the
  shell's password dialog would replace the chooser, answering the app
  "canceled"), and there is no "Connect to Server" (it needs the same
  questions); servers connected in Files show as network places.
- Eject: a drive or network place that the volume monitor has mounted
  there (a mount whose root is that folder, FUSE ones included) has an
  eject button (`media-eject-symbolic`) at the end of its row; clicking
  it ejects the drive, or unmounts it when it cannot be ejected, through
  the shell's removable-media code (as the dock's and the notification's
  Eject do), and the place goes when the mount does.
- The list: columns Name (a symbolic icon for the type: `folder-symbolic`
  for folders, else the generic icon of the type guessed from the name,
  e.g. `image-x-generic-symbolic`), Size (files only, as `g_format_size`
  writes it, right-aligned; not shown when `show-size-column` is false),
  Type (with `show-type-column`, as GTK 4's: `type-format` `category`
  names a broad category from the type's generic icon: Program, Audio,
  Font, Image, Archive, Markup, Text, Video, Contacts, Calendar,
  Document, Presentation, Spreadsheet, else Unknown, and folders
  "Folder"; `description` the type's description, `mime` its MIME type)
  and Modified (the time for today, as `clock-format` asks, "Yesterday",
  the day and month this year, else with the year; with `date-format`
  `with-time` the time follows the date: "Yesterday, 14:05",
  "2 Nov, 14:05", "7 Mar 2025, 10:00"); in Recent and search
  results a Location column (the folder, `~` for the home folder) takes
  the Size column's place. Long names are shortened in the middle,
  keeping their end. Columns go when the list is too narrow (Size or
  Location first, then Type). Folders come first (`sort-directories-first`), sorted
  by the column header clicked (name, size, type, as shown, or modified; clicking the same
  header again flips the order; Recent is always newest first); names
  sort naturally (`file2` before `file10`, case and accents ignored, as
  `g_utf8_collate_key_for_filename` orders them), ties by name. The sort
  and hidden files are GTK's settings
  (`contracts/gsettings/org.gtk.gtk4.Settings.FileChooser.md`). Hidden
  entries (names starting with `.` or ending with `~`) are not listed
  unless `show-hidden`. Files are listed when the mode lists files (not
  when choosing folders or for `SaveFiles`) and pass the current file
  type (globs ignore case; MIME types by the type guessed from the name).
  Only the rows that show are drawn; the list scrolls with the wheel or
  touchpad (three rows a notch) and has a scrollbar that can be dragged
  or clicked.
- Grid (`org.gtk.gtk4.Settings.FileChooser view-type` `grid`; the switch
  in the path bar's row, Ctrl+1 the list, Ctrl+2 the grid, as in GNOME
  Files, writes the key): the same rows, as cells in lines (as many
  120-pixel cells as fit, sharing the width), each with a thumbnail of
  80 × 80 (scaled down to fit, never up, its pixels the screen's) or
  else the theme's full-color icon for the type (`folder` for folders,
  the type's icon names from GIO, else `text-x-generic`; 64 × 64), and
  the name centered below on at most two lines (a longer one shortened
  in the middle of the second line). The column headers stay above the
  grid as its sort buttons. Up and Down move a line, Left and Right a
  cell (mirrored right to left), Page Up and Page Down by the lines that
  show; selection, the mouse and type-ahead are as in the list. Only
  the lines that show are drawn. *GNOME does not document this; Helix's
  choice.*
- Thumbnails (grid): asked for only for the files on screen (not
  folders or empty files), whenever those change, by worker threads
  (two), most wanted first; a request replaces the earlier ones not
  started yet, so scrolling past a folder makes nothing for the files
  scrolled by. For each file, per the freedesktop thumbnail
  specification: the thumbnail in `$XDG_CACHE_HOME/thumbnails/<flavor>/<md5
  of the file's URI>.png` is used when it is fresh (its `Thumb::MTime`
  is the file's modification time, and its `Thumb::URI`, when present,
  the file's URI; without `Thumb::MTime`, when not older than the
  file), first from the smallest flavor at least as big as needed
  (`normal` 128, `large` 256, `x-large` 512, `xx-large` 1024), then
  Helix's own cache (`$XDG_CACHE_HOME/helix/thumbnails/`, 256). A
  picture Helix decodes (as for previews, at most 32 MiB) without one
  is scaled down here and, when larger than the flavor, written to the
  shared cache as `normal` (needed up to 128) or `large` (above), with
  `Thumb::URI`, `Thumb::MTime`, `Thumb::Size`, `Thumb::Mimetype` and
  `Software` (`Helix`), the folder made `0700` and the file `0600`,
  written whole under another name then renamed in, so GNOME's file
  manager and later dialogs use it. Other types get a smaller cached
  thumbnail if there is one, else the installed thumbnailers make one as
  for previews (sandboxed, into Helix's own cache). Files in the
  thumbnail caches themselves get none. In memory the dialog keeps at
  most 48 MiB of thumbnails, those shown least recently going first; a
  change of scale or text size makes them again at the new size.
  *Helix's choice:* the thumbnails are made by the shell, as GNOME's
  apps make them themselves; the `org.freedesktop.thumbnails.Thumbnailer1`
  D-Bus service is not used. With nothing to show: "Folder is empty", "No recent files",
  "No results found", or "This folder cannot be shown: <error>".
- Reading folders: a folder is listed on a worker thread from its
  entries' metadata only (no file is opened or read; the type comes from
  the name, as GTK's `standard::fast-content-type`), so unreadable files
  are listed and a slow drive does not hold the dialog up. Going to a
  folder empties the list at once (the path bar already shows the new
  folder); after 200 ms without an answer the list says "Loading…" with a
  spinner. The entries come in batches as they are read, every 50 ms:
  each batch's rows are merged into those shown where the sort puts
  them (the sort keys are worked out once per entry), hidden files and
  the file type filter apply to them as they come, the current row and
  picked files stay (while the user has chosen none, the first row stays
  the current one; a folder to select on coming back up is selected when
  it arrives, unless the user chose meanwhile), and once the list was
  scrolled, the row at its top stays where it is. While more is coming
  the warning line reads "Loading…" (after the first 200 ms). The places,
  Cancel and every other part work while a folder loads; going elsewhere
  stops the reading (the worker checks between batches), and a listing
  that comes after the user went elsewhere is dropped. Recent, search
  results, the Trash and Other Locations come whole. *GNOME does not
  document this; Helix's choice.*
- Kinetic scrolling: touchpad (finger) scrolling moves the list with the
  fingers; when they lift (the compositor's axis stop), the list goes on
  at the speed of the last 100 ms of movement (none when the fingers
  rested more than 60 ms before lifting, or below 120 pixels a second;
  at most 12000), slowing down as GTK's kinetic scrolling does (speed
  falling as e^(−4t), so a fling goes a quarter of its speed in pixels),
  a step every 16 ms, until below 15 pixels a second or at either end of
  the list. A wheel notch, a press, a key, scrolling the places or
  touching again stops it. *GNOME does not document this; Helix's
  choice.*
- Recent lists recently used files (`$XDG_DATA_HOME/recently-used.xbel`),
  newest first by when they were last modified there (else added), at
  most 100, only local files that still exist and pass the current file
  type. It lists nothing while `org.gnome.desktop.privacy
  remember-recent-files` is false or `recent-files-max-age` is 0, and
  with a positive age only files used within that many days.
- Previews: while the selected file is of a type that can be previewed
  (from its name: pictures — PNG, JPEG, WebP, JPEG XL or SVG —, text, or a
  type an installed thumbnailer handles), a pane right of the list shows
  it, decoded off the UI thread for that file only:
  - a picture scaled down to fit the pane (not above 32 MiB);
  - text: its first lines (of the first 8 KiB), tabs as four spaces;
  - other files: their thumbnail, from the freedesktop thumbnail cache
    (`$XDG_CACHE_HOME/thumbnails/{large,x-large,xx-large,normal}/<md5 of
    the URI>.png`, used when fresh as for the grid's thumbnails), else
    Helix's own cache
    (`$XDG_CACHE_HOME/helix/thumbnails/`, same names), else made into
    Helix's cache by the thumbnailer whose `.thumbnailer` file (in
    `thumbnailers/` under the XDG data folders; skipped when its `TryExec`
    is missing) lists the type, run with `%i` / `%u` (the file's path /
    URI), `%o` (the output) and `%s` (256), and stopped after 10 seconds.
    As in GNOME, the thumbnailer runs in bubblewrap (started like
    extension sandboxes, `contracts/extensions/README.md`): no network
    or other namespaces shared, no user namespaces of its own, a new
    session, a cleared environment (`PATH`, `HOME=/tmp`,
    `GIO_USE_VFS=local`, `LANG`), `/usr` and `/etc` read-only, the
    user's fonts read-only, the file read-only as
    `/tmp/helix-thumbnail-input.<extension>` (that is what `%i` and `%u`
    name) and an empty new folder as `/tmp/helix-thumbnail-output`, the
    only place it can write (`%o` is `thumbnail.png` in it). Only a
    regular PNG file left there is taken. Without bubblewrap no
    thumbnailer runs.
    Thumbnailers' results stay in Helix's own cache (they lack the
    metadata the shared cache's entries need); the grid writes the
    pictures' thumbnails it makes into the shared cache (see
    "Thumbnails").

  When it cannot be shown the pane reads "No preview".
- File type: a button with the current type's name and a menu above it
  listing every type (the current one checked); choosing one filters the
  list at once (a search runs again). It is shown when there are types
  and files are listed.
- Mouse: a press on a row selects it; a second press on the same row
  within `org.gnome.desktop.peripherals.mouse double-click` milliseconds
  activates it (a folder opens; a file is chosen, or when saving, saved
  over after asking). With `multiple`: a click selects one file, Ctrl
  toggles one, Shift picks the range from the last one clicked.
- Keyboard: Tab and Shift+Tab move between the name field (saving), the
  path bar's field (when open), the list, the check box, the file type
  button, Cancel and the accept button. In the list: Up, Down, Page Up,
  Page Down, Home and End move (with Shift, picking the range with
  `multiple`); Enter activates the row (when saving, Enter always saves
  under the name typed); Backspace goes to the parent folder (with the
  folder left selected); Space picks the row (with `multiple`); Ctrl+A
  picks every file (with `multiple`); typing selects the first row whose
  name starts with what was typed (case and accents ignored; typing the
  same letter again goes to the next one; forgotten after 1.5 seconds),
  shown in a bubble; `/` or `~` open the location field with it.
  Anywhere: Alt+Up the parent folder, Alt+Left and Alt+Right back and
  forward, Alt+Home the home folder, Ctrl+L or Alt+D the location field
  (with the folder shown), Ctrl+H hidden files on or off (stored in
  `show-hidden`), Ctrl+1 the list and Ctrl+2 the grid (stored in
  `view-type`), Ctrl+F search (opening files), Escape closes the
  location, search or new folder field, else cancels. In the name and
  path bar fields: Left, Right, Home, End (with Shift selecting),
  Backspace, Delete, Ctrl+A, and typing replaces the selection; copy,
  cut and paste work as in the shell's other fields. Enter is taken only
  once the dialog has been up for 300 ms. On the file type button, Space
  or Down opens its menu (Up, Down, Enter, Escape).
- Opening: the accept button chooses the selected file (with `multiple`,
  the files picked); with a folder selected it opens the folder (as GTK
  does). Choosing a folder: the accept button chooses the folder selected
  in the list, else the folder shown.
- Location field (Ctrl+L): a path (relative to the folder shown, absolute
  or starting with `~`) opens a folder or chooses a file (opening), or
  when saving goes to its folder with its name in the Name field;
  otherwise the warning "“<text>” was not found.".
- Search (opening files; Ctrl+F or the search button): Enter searches the
  folder shown and the folders below it (see "Searching files" below);
  the results show with their Location; Escape (or the search button)
  goes back to the folder.
- Saving: the Name field is at the top with the base name selected (up to
  its last `.`, so typing replaces "Untitled document" and keeps ".odt").
  Clicking a file puts its name there. The accept button (and Enter)
  saves under that name in the folder shown (in Recent, in the selected
  file's folder): an empty name gives the warning "Type a name for the
  file.", a name with `/` is a path as above (a missing folder: "That
  folder does not exist."), a folder's name opens it. An existing file
  needs confirming over the dialog: "Replace File?", "A file named
  “<name>” already exists in “<folder>”. Replacing it overwrites its
  contents.", buttons "Cancel" (back to the chooser) and "Replace" (the
  default). `SaveFiles`: the accept button answers each of `files`'
  names in the folder shown (existing files are not asked about).
- New Folder (saving and choosing folders): a field in the path bar's
  place, "New folder name"; Enter makes the folder in the folder shown
  (on a worker thread) and goes into it. Warnings: "Type a name for the
  folder.", "Folder names cannot contain “/”.", "That name cannot be
  used." (`.` and `..`), "A file or folder with that name already
  exists.", "The folder could not be made: <error>".
- With `org.gnome.desktop.lockdown` `disable-save-to-disk` true, `SaveFile`
  and `SaveFiles` answer 2 without a dialog.
- Results: `uris` as `file:` URIs; `choices` with every choice's value
  (the check box's as `true`/`false`); `current_filter` when there are
  filters.
- Accessibility (`org.a11y.atspi.md`): one dialog object, `file-chooser`,
  for the dialog's whole life, with the path bar (a tool bar of buttons,
  the folder shown toggled), the places (a list box), the file list (a
  list box named "Files in <folder>", "Recent Files", "Starred Files" or "Search results
  for “<text>”", "Trash", "Other Locations"; the grid's cells are its
  rows; every row while there are at most 1000, else those near
  the screen; each row named by the file's name, described "Folder" or
  its size and type, then "Modified <when>" and in Recent and search results its
  folder, with its place among all rows (`posinset`/`setsize`); the
  current row has the focus while the list does), a live status
  "<folder>, <n> items" (or "Loading…") read out when the folder
  changes, the "Grid View" button (toggled while the grid shows), the
  eject buttons ("Eject <drive>"), the fields, the file type (a combo box), and the buttons. Not
  yet tried with Orca (helixdesktop/desktop#71).
- Not offered (helixdesktop/desktop#118), *Helix's choice:*
  - Starring and unstarring in the chooser (GTK's chooser has neither):
    its Starred place is read-only (see Starred above).
  - The window's position (`window-position`): the dialog is the
    shell's, centered over everything with the screen dimmed, and cannot
    be moved, so there is no position to remember (GTK 4 on Wayland
    cannot place its window either).
  - "Connect to Server", and unlocking or answering questions while
    mounting from Other Locations (see there).

### Searching files

The shell runs searches off the UI thread. When a file index runs on
the session bus (localsearch, `org.freedesktop.LocalSearch3`, else
Tracker 3's `org.freedesktop.Tracker3.Miner.Files`; neither is started
for a search), the chooser asks it first, through its SPARQL endpoint
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
