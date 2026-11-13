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
| `org.helix.Extensions1.xml` | the shell, on the session bus | `/org/helix/Extensions1` (bus name `org.gnome.Shell`) |

In item paths, `<id>` is the item's id with `-` written as `_` (D-Bus
object paths cannot hold `-`).

This document is API version **1.0**. Minor versions only add
(properties, methods, contribution kinds, raised limits); nothing in 1.x
is removed or changes meaning.

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

[settings]
minutes = { type = "int", default = 0, min = 0, max = 480, label = "Minutes", help = "0: until turned off" }
```

- `id`: two or more dot-separated parts of `A-Z a-z 0-9 _ -`, at most
  128 characters, equal to the directory's name.
- `api`: `"1.N"`; Helix runs it when it supports API 1 at minor N or
  later (this version: 1.0).
- `exec`: a relative program is resolved in the extension's directory.
- `activation`: `onStartup` (when the session starts, or when the
  extension is enabled) or `onVisible` (when one of its items is first
  shown: the panel item is in the layout, or quick settings open). Its
  items are drawn from the manifest until it reports.
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
- An enabled extension runs only when its permissions are
  **approved**: the manifest's permissions are a subset of those
  recorded for it in `approved`. Enabling an extension from the
  Extensions dialog or `helix-extensions enable` approves its current
  permissions. An update that asks for more waits, shown as "needs
  approval", until approved again. An extension without permissions
  needs no approval.
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

| Name | Allows | Shown as |
|------|--------|----------|
| `notifications` | `Host.ShowNotice` | show notifications |
| `inhibit-idle` | `Host.InhibitIdle` | keep the screen on |
| `launch-apps` | `Host.Launch` | start apps |

A call needing a permission the extension was not granted fails with
`org.freedesktop.DBus.Error.AccessDenied`. These permissions limit what
an extension can ask of the shell. In API 1 an extension is an ordinary
program of the user: the shell does not sandbox what it does on its own
(see the proposal).

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

## Menus (`org.helix.Extension1.Menu`)

- `Layout` (`a(iissbb)`): the entries, each `(id, parent, kind, label,
  enabled, checked)`. `parent` is 0 for the top level, else a
  `submenu` entry's id. `kind` is `item`, `check`, `radio`,
  `separator`, `heading` (a label that cannot be chosen) or `submenu`.
  Entries come in their order; ids are positive and unique.
- Choosing an entry calls `Activated(id)` and closes the menu. Menus
  look and behave like the panel's other menus.

## Session bus: `org.helix.Extensions1`

For tools (`helix-extensions`) and the Extensions dialog's state:

- `ListExtensions() → a{sa{sv}}`: by id, `name`, `description`,
  `version`, `path`, `type` (`system`/`user`/`development`), `enabled`
  (`b`), `state` (`s`), `error` (`s`, the reason when invalid or
  crashed), `permissions` (`as`), `approved` (`b`).
  - `state` is one of `running`, `starting`, `stopped`, `disabled`,
    `invalid`, `needs-approval` or `crashed`.
- `Enable(id, approve)`: enables it. With `approve` true, also approves
  its current permissions.
- `Disable(id)`, `Restart(id)`.
- `GetLog(id) → as`: its last log lines.
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
- Signal `Changed(id)`: an extension's entry changed.

## The Extensions dialog

The panel's right-click menu has "Extensions…" under "Customize
Panel…". It lists the extensions (development ones last). Each entry
shows its name, version and description, and its state. It has an on/off
switch, which enables it and approves its permissions, or disables it.
It lists what it can do ("Can: keep the screen on"). When it is invalid
or crashed, it shows the reason or its last log line.

When it declares settings, a "Settings" button shows them below it
(sorted by key):

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
- `enable [--yes] ID`: shows the permissions and asks before approving
  them (unless `--yes`);
- `disable ID`, `restart ID`;
- `logs ID`;
- `set ID KEY VALUE`: `true`/`false`, numbers, text; a string-list as
  comma-separated text;
- `install PATH`: copies a directory, or unpacks a `.tar`, `.tar.gz` or
  `.tar.xz` archive holding one directory, into the per-user directory,
  after checking its manifest (replacing an older copy);
- `remove ID`: deletes a per-user extension;
- `run PATH`: runs a development extension (`RunDevelopment`), prints
  its log as it grows, and stops it on Ctrl+C.

All but `install`, `remove` and `list`/`info` (which then read the
directories) need a running Helix shell.
