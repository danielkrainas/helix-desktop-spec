# Helix extensions, API 1

The contract for Helix's own extensions (the design and its reasons are
in `proposals/0001-extensions.md`). An extension is a program that adds
items to defined places: panel items, quick settings toggles and
sliders, and their menus. It runs in its own process and talks to the
shell over a private D-Bus connection, through the interfaces in this
directory:

| File | Served by | Object |
|------|-----------|--------|
| `org.helix.Extension1.Host.xml` | the shell | `/org/helix/Extension1/Host` |
| `org.helix.Extension1.PanelItem.xml` | the extension | `/org/helix/Extension/panel_item/<id>` |
| `org.helix.Extension1.Toggle.xml` | the extension | `/org/helix/Extension/toggle/<id>` |
| `org.helix.Extension1.Slider.xml` | the extension | `/org/helix/Extension/slider/<id>` |
| `org.helix.Extension1.Menu.xml` | the extension | any path the extension names |
| `org.helix.Extension1.DockItem.xml` (1.1) | the extension | `/org/helix/Extension/dock_item/<id>` |
| `org.helix.Extension1.Commands.xml` (1.1) | the extension | `/org/helix/Extension/commands` |
| `org.helix.Extensions1.xml` | the shell, on the session bus | `/org/helix/Extensions1` (bus name `org.gnome.Shell`) |

In item paths, `<id>` is the item's id with `-` written as `_` (D-Bus
object paths cannot hold `-`).

This document is API version **1.2**. Minor versions only add
(properties, methods, contribution kinds, raised limits); nothing in 1.x
is removed or changes meaning. What 1.1 and 1.2 added is marked "(1.1)"
or "(1.2)"; a manifest asking for an earlier `api` cannot use it (those
manifest entries are ignored).

## Extension directories

An extension is a directory named by its id, holding `extension.toml`
and its files. Helix looks in `$XDG_DATA_HOME/helix/extensions/` (per
user), then in each `$XDG_DATA_DIRS` entry's `helix/extensions/`
(system). A per-user extension hides a system one with the same id.

## Manifest (`extension.toml`)

```toml
id = "org.example.Caffeine"        # = the directory's name
name = "Caffeine"                  # at most 64 characters
description = "Keep the screen on." # at most 256 characters
version = "1.4.0"                  # the extension's own, free form (64)
api = "1.0"                        # the API version it needs
exec = ["bin/caffeine", "--quiet"] # program (relative to the directory, or absolute) and arguments
license = "MIT"                    # optional
url = "https://example.org"        # optional
activation = ["onStartup"]         # optional, default ["onStartup"]
permissions = ["inhibit-idle"]     # optional, default []
enabled-by-default = false         # optional; system extensions only
sandbox = true                     # optional (1.2): run in a sandbox

[[contributes.panel-item]]
id = "indicator"                   # [a-z0-9-], at most 32; unique in the extension
name = "Caffeine"                  # shown in the editors (64)
zones = ["right"]                  # optional: left, center, right (default all)
icon = "my-cup-symbolic"           # optional default icon
label = ""                         # optional default label

[[contributes.quick-toggle]]
id = "toggle"
title = "Caffeine"
icon = "my-cup-symbolic"           # optional

[[contributes.quick-slider]]
id = "strength"
title = "Strength"
icon = "my-cup-symbolic"           # optional

[[contributes.dock-item]]           # (1.1)
id = "stack"
name = "Downloads Stack"           # shown in its menu and tooltip
icon = "folder-download-symbolic"  # optional
accepts-drops = true               # optional: files may be dropped on it

[[contributes.command]]             # (1.1)
id = "toggle"
title = "Toggle Caffeine"          # at most 64 characters
keybinding = "<Super><Shift>c"     # optional: a suggestion

[settings]
minutes = { type = "int", default = 0, min = 0, max = 480, label = "Minutes", help = "0: until turned off" }
```

- `id`: two or more dot-separated parts of `A-Z a-z 0-9 _ -`, at most
  128 characters, equal to the directory's name.
- `api`: `"1.N"`; Helix runs it when it supports API 1 at minor N or
  later (this version: 1.2).
- `exec`: a relative program is resolved in the extension's directory.
- `activation`: `onStartup` (when the session starts, or when the
  extension is enabled) or `onVisible` (when one of its items is first
  shown: the panel item is in the layout, or quick settings open). Its
  items are drawn from the manifest until it reports. (1.1)
  `onDBusName:<name>`: when bus name `<name>` gets an owner on the
  session bus (it is not stopped when the name goes away). Running one
  of its commands always starts an extension.
- (1.1) A panel item may say `popover = true`: its primary click shows
  the extension's own panel (see "Popovers"). It needs the `popover`
  permission.
- `permissions`: see "Permissions"; unknown names make the manifest
  invalid.
- Settings `type`: `bool`, `int`, `double`, `string`, `enum` (with
  `choices`, a list of strings) or `string-list`. Each needs a `label`
  and a `default` of its type (an `enum` default is one of its
  choices); `min`/`max` apply to `int` and `double`; `help` is optional.
- Unknown top-level names and table entries are ignored (later minor
  versions may add some).

A manifest that cannot be read or breaks these rules makes the extension
**invalid**: it is listed with the reason, and never started.

## Enabling, approving and placing

Kept in `org.helix.extensions` (`contracts/gsettings/org.helix.extensions.md`):

- An extension is **enabled** when its id is in `enabled`, or it is a
  system extension with `enabled-by-default` and its id is not in
  `disabled`. `disable-all` turns all extensions off.
- **Holding Shift at login** turns all extensions off for that
  session, without changing any setting. When the shell first starts,
  the compositor reads which keys each keyboard holds (`EVIOCGKEY` on
  the seat's input devices; on the DRM backend only). If a Shift key is
  down, it starts the shell (and any restart of it in that session) with
  `HELIX_EXTENSIONS_OFF=1`. The shell then starts no extension and
  shows none of their items. A notification says "Extensions are off
  for this session" ("Shift was held at login."), and the Extensions
  dialog lists them as `disabled`, with "Off for this session" as the
  reason. Turning one on there waits for the next session. (Nested
  sessions: `HELIX_TEST_SHIFT=1` in the compositor's environment does
  the same, for testing.)
- An enabled extension runs only when its permissions are
  **approved**: the manifest's permissions are a subset of those
  recorded for it in `approved`. Turning an extension on in the shell's
  own Extensions dialog approves its current permissions (the dialog
  shows them next to the switch). Any other program (Settings,
  `helix-extensions`) asks through `Enable` on the session bus, and the
  shell then asks the user first (see "Approval prompt"). An update that
  asks for more waits, shown as "needs approval", until approved again.
  An extension without permissions needs no approval.
- The first time an extension is enabled, its panel items are added to
  the end of the first zone they suit (`org.helix.shell`
  `panel-left`, `panel-center`, `panel-right`; in the right zone, before
  `helix:status` and `helix:power` when they are there), and its toggles and
  sliders to the end of `quick-settings`, as `<extension id>/<item id>`.
  Their ids are then added to `placed`, so items the user removes later
  are not added again.

## Processes and connections

- The shell starts the program with the extension's directory as its
  working directory, the environment of the session plus
  `HELIX_EXTENSION_ID` (its id) and `HELIX_EXTENSION_FD` (`3`). File
  descriptor 3 is one end of a Unix socket pair; the shell holds the
  other. Both sides speak D-Bus over it, peer to peer: the shell as the
  server (it sends the GUID), the extension as the client. Nothing is
  on the session bus, so no other process can pose as either side.
- When the systemd user manager is reachable (`$XDG_RUNTIME_DIR/systemd/private`
  exists), the program runs through `systemd-run --user --scope` in a
  scope named `app-helix\x2dextension-<id with . as _>-<pid of the shell>-<n>.scope`
  with `MemoryMax=256M` and `CPUWeight=50`. Otherwise it is started
  directly.
- Its standard output and error, and its `Log` calls, go to its log: the
  last 200 lines are kept, and appended to
  `$XDG_STATE_HOME/helix/extensions/<id>.log` (cut to its last half when
  over 256 KiB).
- The first call must be `Host.Hello`. Before it, every other call
  fails with `org.freedesktop.DBus.Error.AccessDenied`. The shell reads
  the extension's items after `Hello` returns.
- **Stopping**: when the extension is disabled or the session ends, the
  shell closes the connection and sends `SIGTERM`; after 2 seconds,
  `SIGKILL`.
- An extension is started once at a time: a start in progress, or a
  restart waiting (see "Robustness"), is not started again. If the
  connection cannot be set up, or `Hello` does not come in time, the
  process is killed, and that counts as one crash.

## Robustness

- The shell never waits for an extension. Its calls to the
  extension's items time out after 250 milliseconds; a late answer is
  ignored. Three timeouts in a row count as a crash.
- **Crashes**: when the program exits without being asked, or the
  connection breaks, its items show their manifest defaults, dimmed, and
  the shell starts it again after 1, then 5, then 30 seconds (30 from
  then on). After 5 crashes within 10 minutes it is not restarted again
  in this session ("crashed"), and a notification says "<name> stopped
  working" with its last log line (its own output, not the shell's
  notes such as "exited"). Turning it off and on again starts it
  afresh.
- **Content limits** (more is cut off; a bad value is ignored and
  logged):
  - labels: 64 characters in the panel, 128 elsewhere;
  - tooltips: 512 characters, 8 lines;
  - `Graph`: the last 64 values;
  - `Meter`, `Graph`, `Value`: clamped to 0–1 (`Meter` below 0 means
    none);
  - `IconData`: PNG data up to 256×256 pixels;
  - `Icon`: an icon theme name, or a file path inside the extension's
    directory (other paths are ignored);
  - menus: 64 entries per level, 2 levels, 256 entries in all;
  - `Hello` must come within 10 seconds of starting, or the start counts
    as a crash.
- Items whose ids the manifest does not declare are not shown.
- Nothing from extensions is shown while the session is locked.

## Permissions

API permissions limit what an extension can ask of the shell:

| Name | Allows | Shown as |
|------|--------|----------|
| `notifications` | `Host.ShowNotice` | show notifications |
| `inhibit-idle` | `Host.InhibitIdle` | keep the screen on |
| `launch-apps` | `Host.Launch` | start apps |
| `popover` (1.1) | panel items with `popover = true` | show its own panels |

A call needing a permission the extension was not granted fails with
`org.freedesktop.DBus.Error.AccessDenied`.

(1.2) System permissions say what a sandboxed extension may reach on its
own (they mean nothing without `sandbox = true`, and need `api = "1.2"`):

| Name | Gives the sandbox | Shown as |
|------|-------------------|----------|
| `network` | the network | use the network |
| `files:home` | the home folder, writable | read and change all your files |
| `files:documents`, `files:downloads`, `files:music`, `files:pictures`, `files:videos`, `files:desktop` | that XDG folder, writable | read and change your Documents (and so on) |
| `session-bus` | the session bus (all of it) | talk to your other programs |
| `system-bus` | the system bus (all of it) | talk to system services |

## Sandboxes (1.2)

An extension with `sandbox = true` runs under bubblewrap (`bwrap`), as a
Flatpak app runs. It sees:

- the system's `/usr` (with `/bin`, `/sbin`, `/lib*` as on the host,
  whether directories or links into `/usr`) and `/etc`, read-only;
- its own directory, read-only, as its working directory;
- a home of its own, writable: `$XDG_DATA_HOME/helix/extension-data/<id>`
  (created when needed), as `HOME`;
- a private `/tmp`, `/proc`, a minimal `/dev`, and an empty
  `$XDG_RUNTIME_DIR` holding only the Wayland socket when it has the
  `popover` permission;
- nothing else of the user's files, no network and no D-Bus buses,
  except what its system permissions give;
- its descriptor 3 (the connection to the shell), as without a sandbox.

All namespaces are new (user, mount, PID, IPC, UTS, cgroup, and the
network one unless `network`); it ends with the shell (`--die-with-parent`)
and gets its own terminal session (`--new-session`). Without `bwrap`
installed, a sandboxed extension does not start (state `invalid`,
"needs bubblewrap (bwrap)").

`org.helix.extensions` `require-sandbox` (an administrator's lock):
while true, extensions without `sandbox = true` do not run (state
`disabled`, "Only sandboxed extensions may run"); development
extensions too.

The Extensions dialog says "Sandboxed" for sandboxed extensions, else
"Full access: it can do anything you can".

## The host (`org.helix.Extension1.Host`)

- `Hello(api_version) → (api_version, permissions, info)`:
  `api_version` is the `"1.N"` the extension uses. It fails with
  `org.freedesktop.DBus.Error.NotSupported` when Helix does not support
  it. The reply's version is the one in use, its permissions those
  granted, and `info` holds `helix-version` (`s`) and `locale` (`s`).
- `GetSetting(key) → value`: the value, or the default. Unknown keys
  fail with `org.freedesktop.DBus.Error.InvalidArgs`.
- `SetSetting(key, value)`: the value must have the key's type
  (`b`, `x`, `d`, `s`, `s` for `enum` (one of its choices), `as`) and
  be within `min`/`max`, else `InvalidArgs`. Settings are kept in
  `$XDG_CONFIG_HOME/helix/extensions/<id>.toml`.
- `SettingChanged(key, value)`: a setting changed (by the extension,
  the Extensions dialog, or `helix-extensions set`).
- `Log(level, message)`: level 0 error, 1 warning, 2 info, 3 debug.
- `ShowNotice(title, body, icon)` (`notifications`): a notification from
  the extension's name.
- `InhibitIdle(reason) → handle` (`inhibit-idle`): keeps the session
  from going idle (`org.gnome.SessionManager.Inhibit`, idle flag) until
  `Uninhibit(handle)` or the extension stops.
- `Launch(desktop_id, uris)` (`launch-apps`): starts an app by its
  desktop file id, with optional URIs.

## Panel items (`org.helix.Extension1.PanelItem`)

- **Properties**:
  - `Icon` (`s`), `IconData` (`ay`, a PNG, preferred when not empty);
  - `Label` (`s`);
  - `Tooltip` (`s`): its first line is shown in bold;
  - `AccessibleName` (`s`);
  - `Visible` (`b`);
  - `Attention` (`b`): an accent line under the item;
  - `Meter` (`d`): a bar under the item, 0–1; below 0, none;
  - `Graph` (`ad`): a sparkline of recent values, 0–1, oldest first;
  - `Menu` (`o`): a `Menu` object, `/` for none.
- **Drawing**: the icon (16 pixels; symbolic icons in the panel's text
  color), then the label, then the graph (32 pixels wide). The meter is
  a 2-pixel bar along the item's bottom. An item with none of these, or
  not `Visible`, is not shown.
- **Clicks**:
  - A primary click calls `Activate(x, y)`. If that fails and the item
    has a menu, the menu opens below the item.
  - A secondary click opens the menu if there is one.
  - A middle click calls `SecondaryActivate(x, y)`.
  - Scrolling calls `Scroll(delta, orientation)`, 120 per wheel step.

  `x` and `y` are the item's bottom-left corner in the layout's
  coordinates.
- **Tooltip**: resting the pointer on the item for 0.6 seconds shows
  `Tooltip` as the status icons' tooltips are shown.
- Keyboard focus (`switch-panels`) reaches it; Enter acts as a primary
  click.

## Popovers (1.1)

A panel item with `popover = true` (and the `popover` permission) shows
the extension's own panel when clicked, drawn by the extension with any
toolkit.

- A primary click calls the item's `TogglePopover(x, y, output)`
  (`org.helix.Extension1.PanelItem`) instead of `Activate`. `x`, `y` are
  where the popover's top-left corner goes, in the coordinates of the
  monitor whose connector is `output`: under the item, moved left so
  that 480 pixels fit. The extension opens its popover there, or closes
  it if it is open.
- The popover is a layer surface (`zwlr_layer_shell_v1`) on that
  output: layer `overlay`, namespace `helix-extension-popover`,
  anchored top and left with those margins, and exclusive zone -1 (so
  the margins count from the monitor's edges, not from below the
  panel). It may take the keyboard
  (`exclusive` or `on_demand`). It is at most 480 by 640 pixels; a
  larger one is closed.
- The compositor closes it (the layer surface's `closed` event) when the
  user clicks outside it, or when another popover opens. Closing it on
  Escape is the extension's part.
- While the session is locked, no popover is shown (they are closed at
  locking).

## Quick settings (`org.helix.Extension1.Toggle`, `org.helix.Extension1.Slider`)

- A **toggle** is a switch in the quick settings grid, at its place in
  `quick-settings`.
  - Properties: `Title`, `Subtitle`, `Icon` (`s`), `Active`,
    `Sensitive` (`b`), `Details` (`o`, a `Menu` or `/`).
  - It shows its title, with its subtitle under it when not empty, and
    is highlighted while `Active`; it is dimmed while not `Sensitive`
    or while its extension is not running. With `Details`, a "›" at its
    right opens that menu, in place of quick settings.
  - A click calls `SetActive(!Active)` (not when not `Sensitive`). The
    switch changes only when `Active` does.
- A **slider** is a row under the built-in sliders, in the order of
  `quick-settings`.
  - Properties: `Title`, `Icon` (`s`), `Value` (`d`, 0–1), `Sensitive`
    (`b`).
  - A click on its bar calls `SetValue` with the value at that point.

## Dock items (`org.helix.Extension1.DockItem`) (1.1)

- Shown in the dock after the drives and before the Trash, in the order
  of the enabled extensions (by id) and their manifests, while the dock
  shows those (`show-mounts`/`show-trash` do not apply to them).
- **Properties**: `Icon` (`s`), `IconData` (`ay`), `Label` (`s`, its
  name in tooltips; the manifest's `name` when empty), `Running` (`b`,
  the running dot), `Count` (`x`, a badge when above 0), `Progress`
  (`d`, a bar, 0–1; below 0, none), `Urgent` (`b`, as urgent apps are
  shown), `Menu` (`o`, `/` for none).
- **Clicks**:
  - A click calls `Activate()`.
  - A secondary click opens its menu, if it has one, as dock menus open.
- **Drops**: with `accepts-drops`, files dropped on it (`text/uri-list`)
  call `Drop(uris)`.
- While its extension is not running it is drawn from the manifest,
  dimmed.

## Commands (`org.helix.Extension1.Commands`) (1.1)

- Each `[[contributes.command]]` is a command `<extension id>/<command
  id>`. Command ids follow the rules of item ids, but are their own: a
  command may share an id with an item.
- **Keys**: `org.helix.extensions` `command-keys` maps commands to
  accelerators (GNOME's syntax; empty: none). The first time an
  extension is enabled, a command's suggested `keybinding` is put there
  if no shell, window manager, custom or other command key uses it.
  Otherwise the command is left without one, and a notification says
  so. The user can change or clear it in the Extensions dialog or with
  `helix-extensions bind`.
- **Running**: a command runs from its key, from the run dialog (Alt+F2:
  typing its title exactly, any case, runs it; Tab completes titles), or
  from `helix-extensions command`. The shell calls `Run(id)` (the
  command's id without the extension's). If the extension is not
  running, but is enabled and approved, it is started first, and
  `Run` is called after its `Hello` (within its 10 seconds). No command
  runs while the session is locked.

## Menus (`org.helix.Extension1.Menu`)

- `Layout` (`a(iissbb)`): the entries, each `(id, parent, kind, label,
  enabled, checked)`. `parent` is 0 for the top level, else a
  `submenu` entry's id. `kind` is `item`, `check`, `radio`,
  `separator`, `heading` (a label that cannot be chosen) or `submenu`.
  Entries come in their order; ids are positive and unique.
- Choosing an entry calls `Activated(id)` and closes the menu. Menus
  look and behave like the panel's other menus.

## Session bus: `org.helix.Extensions1`

For tools (`helix-extensions`, the Settings app's Extensions panel) and
the Extensions dialog's state:

- `ListExtensions() → a{sa{sv}}`: by id, `name`, `description`,
  `version`, `path`, `type` (`system`/`user`/`development`), `enabled`
  (`b`), `state` (`s`), `error` (`s`, the reason when invalid or
  crashed), `permissions` (`as`), `approved` (`b`), (1.1)
  `commands` (`a(sss)`: command id, title, accelerator), and (1.2)
  `sandboxed` (`b`).
  - `state` is one of `running`, `starting`, `stopped`, `disabled`,
    `invalid`, `needs-approval` or `crashed`.
- `Enable(id, approve)`: enables it. With `approve` true and permissions
  not yet approved, the shell shows the approval prompt and answers once
  the user did: allowed, the extension is enabled and its current
  permissions approved; refused (or the prompt could not be shown), the
  call fails with `org.freedesktop.DBus.Error.AccessDenied` and nothing
  changes. With `approve` false, or nothing to approve, it only enables
  (an extension waiting for approval then shows "needs approval").
  Callers should allow for the user's time (no short reply timeout).
- `Disable(id)`, `Restart(id)`.
- `GetLog(id) → as`: its last log lines.
- `GetSettings(id) → a(ssva{sv})`: its settings in the manifest's order:
  key, type (`bool`, `int`, `double`, `string`, `enum`, `string-list`),
  current value (`b`, `x`, `d`, `s`, `s`, `as`), and details: `label`
  (`s`), `help` (`s`, may be empty), `default` (of the setting's
  type, as the value), `min` and `max`
  (`d`, when declared; `int` and `double`), `choices` (`as`, `enum`).
  An unknown id fails with `org.freedesktop.DBus.Error.InvalidArgs`.
- `SetSetting(id, key, value)`: as `Host.SetSetting`, for the
  extension's settings.
- `RunDevelopment(path) → id`: runs the extension in directory `path`
  (its manifest's id need not match the directory's name) as a
  **development** extension. It needs no enabling or approval, and its
  items are shown at the ends of their zones and of quick settings
  without being saved in the layout (after the zone's other items). It
  is restarted when a file in the directory changes (checked every
  second). It stops when the
  caller leaves the bus, or on `StopDevelopment(id)`.
- (1.1) `RunCommand(command)`: runs `<extension id>/<command id>`.
- (1.1) `BindCommand(command, accelerator)`: sets its key (`""`: none).
  An accelerator that does not parse, or one another key uses, fails
  with `org.freedesktop.DBus.Error.InvalidArgs`.
- Signal `Changed(id)`: an extension's entry changed.

## Approval prompt

When `Enable` asks to approve permissions, the shell shows a dialog
over everything (like its other questions):

- title "Turn on <name>?";
- "It will be able to:" and one line per permission it asks for (the
  texts of "Permissions"); for an extension that is not sandboxed, also
  "It is not sandboxed: it can do anything you can.";
- buttons "Cancel" and "Turn On"; Escape cancels, Enter turns it on.
  There is no countdown.

It is refused at once, without being shown, while the screen is locked
or while another of the shell's questions is open. Locking the screen
while it is shown refuses it.

## The Extensions dialog

The panel's right-click menu has "Extensions…" under "Customize
Panel…". When the Settings app is installed (`helix-settings-app` on
`PATH`, from helix-apps), it opens Settings' Extensions panel
(`helix-settings-app extensions`). Otherwise the shell shows its own
dialog, described here.

It lists the extensions (development ones last). Each entry
shows its name, version and description, and its state. It has an on/off
switch, which enables it and approves its permissions, or disables it.
It lists what it can do ("Can: keep the screen on"). When it is invalid
or crashed, it shows the reason or its last log line.

(1.1) Its commands are listed under it with their keys ("None" when
unbound); keys are changed in Settings or with `helix-extensions bind`.

When it declares settings, a "Settings" button shows them below it
(in the manifest's order):

- `bool`: a switch;
- `enum`: a button showing the value, which steps to the next choice;
- `int` and `double`: the value between − and + buttons (steps of 1,
  or a twentieth of `max` − `min`, else 0.1);
- `string` and `string-list`: the value, shown but not editable there
  (`helix-extensions set`).

Escape or "Done" closes it.

## `helix-extensions`

A command for users and developers:

- `list`, `info ID`;
- `enable ID`: `Enable(ID, true)`: the shell asks the user before
  approving anything (see "Approval prompt"); `--yes` is accepted and
  ignored (earlier versions skipped their own question with it);
- `disable ID`, `restart ID`;
- `logs ID`;
- `set ID KEY VALUE`: `true`/`false`, numbers, text; a string-list as
  comma-separated text;
- (1.1) `command COMMAND`: runs `<extension id>/<command id>`;
  `bind COMMAND ACCELERATOR|none`: sets or clears its key;
- `install PATH`: copies a directory, or unpacks a `.tar`, `.tar.gz` or
  `.tar.xz` archive holding one directory, into the per-user directory,
  after checking its manifest (replacing an older copy);
- `remove ID`: deletes a per-user extension;
- `run PATH`: runs a development extension (`RunDevelopment`), prints
  its log as it grows, and stops it on Ctrl+C.

All but `install`, `remove` and `list`/`info` (which then read the
directories) need a running Helix shell.
