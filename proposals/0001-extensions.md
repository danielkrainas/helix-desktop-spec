# 0001 — Helix extensions

- **Status**: Accepted; Phase 0 and API 1.0 (Phase 1) implemented, contract in `contracts/extensions/`
- **Summary**: A way to add things to the Helix desktop: panel items,
  status indicators, quick-settings toggles and sliders, dock items,
  commands and settings pages. An extension declares what it adds in a
  manifest. It runs as its own process and talks to the shell over a
  versioned, documented D-Bus API. It publishes **what** to show (icons,
  labels, menus, values), and the shell draws it. Extensions never run
  inside the shell and never patch it, so a broken or outdated extension
  cannot break the desktop, and a new shell release does not break
  extensions written against an API version it still supports.

## Motivation

People add a fairly small set of things to their desktop:

- a system monitor, weather, a clipboard history or a timer in the panel;
- tray icons for apps (chat, sync clients, VPNs);
- toggles for things the built-in quick settings lack (a VPN, caffeine /
  keep-awake, a game mode, a USB device, a smart light);
- items in the dock (a folder stack, a running service);
- a global shortcut that does something;
- a small preferences page for each of these.

Plasma and Xfce serve this well with **applets and plugins** that fill
defined slots, and that users add, remove and arrange themselves. Their
weakness is that most run **inside the panel process** through a native
plugin ABI, so a crashing plugin takes the panel down with it.

GNOME Shell extensions are the opposite. They can change anything,
because they are JavaScript that patches the shell's internal objects
at run time. That makes them powerful, but:

- There is **no API**. Extensions depend on internals that change every
  release, so most break on each GNOME upgrade, and their authors must
  re-port them twice a year.
- They run **in the compositor process**. An exception or a slow loop in
  one freezes or kills the whole session (on Wayland, logging the user
  out).
- Extensions interfere with each other and with the shell in ways
  nobody can test.

Helix already does not run GNOME Shell extensions
(`contracts/dbus/org.gnome.Shell.Extensions.md`), and supporting them
is out of scope (`SCOPE.md`). This proposal gives Helix its own system.
It takes the **what** from Plasma and Xfce: defined slots for the
things people want. It takes the **how** from VS Code extensions and
XDG portals: a declared manifest, a documented and versioned API,
contribution points instead of patching, and the extension kept out of
the host's process.

## Goals

1. **Cover the common cases**: panel items, indicators, quick-settings
   toggles and sliders, dock items, commands and keybindings, settings.
2. **A real API**: documented in this spec as contracts like the GNOME
   ones, versioned, with compatibility rules, and checked by the same
   conformance tests (`assert_matches_contract`).
3. **Isolation**: each extension in its own process. The shell never
   blocks on an extension, never trusts what it sends, and survives its
   crash, hang or misbehavior.
4. **Consistent look**: the shell draws contributed items with its own
   theme, scale, accessibility and keyboard navigation. Extensions do not
   need to know how.
5. **Any language**: the API is D-Bus, so any language with a D-Bus
   library works. Helix ships a Rust SDK; others are welcome.
6. **The user is in control**: users add, remove and arrange items, see
   what each extension may do, and can turn any extension off.

## Non-goals

- Changing how the shell itself behaves: replacing the overview,
  restyling window frames, changing animations. Those are Helix
  settings and features, or proposals of their own. Extensions add
  things in slots; they do not alter the shell.
- Compatibility with GNOME Shell extensions, or running their code.
- Drawing arbitrary content directly into the panel (see "Popovers" for
  the escape hatch).
- A public extension store in the first version.

## Overview

```
 ┌─────────────── helix-shell ───────────────┐      ┌── extension process ──┐
 │ panel  quick settings  dock  settings UI  │      │  (any language)       │
 │      ▲ draws items from their state       │      │                       │
 │ extension host (in the shell):            │ D-Bus│ exports its items:    │
 │  - reads manifests, starts processes  ────┼─────►│  PanelItem, Toggle,   │
 │  - watches items' properties          ◄───┼──────┤  Menu, DockItem…      │
 │  - serves Host1 (settings, logs, …)   ◄───┼──────┤ calls Host1           │
 └───────────────────────────────────────────┘ p2p  └───────────────────────┘
```

- An **extension** is a directory with a manifest (`extension.toml`) and
  a program.
- The shell's **extension host** reads the manifests. When an
  extension's **activation event** happens, the host starts the program
  as its own process, in its own systemd scope. It hands the program a
  private, peer-to-peer D-Bus connection to the shell. It does not use
  the session bus, so names cannot clash and other processes cannot
  impersonate the shell.
- The extension **exports objects** for what it contributes (a panel
  item, a toggle…), with properties that describe them. The shell reads
  the properties and follows their `PropertiesChanged` signals, as it
  follows StatusNotifierItems or `com.canonical.Unity.LauncherEntry`
  today.
- The user interacts with the item; the shell calls the item's methods
  (`Activate`, `Toggle`, `MenuActivated`…), asynchronously and with a
  deadline.
- The extension calls the shell's **`Host1`** object for services the
  shell provides: its settings, log messages, showing a notice. Each of
  these is gated by the manifest's permissions.

## Manifest

```toml
# ~/.local/share/helix/extensions/org.example.Caffeine/extension.toml
id = "org.example.Caffeine"          # reverse-DNS, = the directory name
name = "Caffeine"
description = "Keep the screen on while you need it."
version = "1.4.0"                    # the extension's own version
api = "1.2"                          # lowest Helix extension API it needs
exec = ["bin/caffeine"]              # relative to the directory, or absolute
license = "MIT"
url = "https://example.org/caffeine"

# When to start the program (see "Activation"); default: on login.
activation = ["onStartup"]

# What the extension may do through the API (see "Permissions").
permissions = ["inhibit-idle"]

[[contributes.quick-toggle]]
id = "toggle"
title = "Caffeine"
icon = "my-caffeine-symbolic"        # icon theme name, or a path in the directory

[[contributes.panel-item]]
id = "indicator"
name = "Caffeine indicator"          # shown in the panel editor
zones = ["right"]                    # zones it suits; default: any

[[contributes.command]]
id = "toggle"
title = "Toggle Caffeine"
keybinding = "<Super><Shift>c"       # a suggestion; the user can change or clear it

[settings]                           # see "Settings"
duration = { type = "int", default = 0, min = 0, max = 480, label = "Minutes (0: until turned off)" }
```

Static parts of an item (its title, default icon, the zones it suits)
are in the manifest. So the shell can show a contributed item in its
editors, and draw it in its last known or default state, before the
extension's process runs. The program supplies the live state.

An extension without a valid manifest, or needing an `api` the shell
does not support, is listed but not started. The settings page says why:
"Needs Helix extension API 2.0; this Helix supports 1.0–1.4".

## Contribution points (API 1)

Each contribution point is a D-Bus interface that the extension
implements, on an object at `/org/helix/Extension/<kind>/<id>`. The id
is the one from the manifest. A full contract (interface XML and
behavior notes) is written for each one when the proposal is accepted;
what follows is the intended shape.

### Content limits

Everything the shell shows from an extension goes through the same
checks. Over-long text is cut off with "…". An icon that is too large,
or a menu that is too deep, is not shown, and the extension gets a
warning in its log. Limits in API 1:

- **Labels**: plain text (no markup), at most 64 characters in the
  panel and 128 elsewhere.
- **Icons**: an icon theme name, a file in the extension's directory,
  or PNG data up to 256×256. Symbolic icons are recolored like the
  shell's own.
- **Menus**: at most 64 items per level and 2 levels.
- **Updates**: properties may change at any rate. The shell picks up at
  most one change per item per frame, and drops excess.

### Panel items (`org.helix.Extension1.PanelItem`)

Panel items are the Plasma/Xfce applet: something in the top panel.

- Properties:
  - `Icon`, `Label`, `Tooltip`;
  - `AccessibleName`, required when there is no label;
  - `Visible`;
  - `Attention`, which the shell highlights like an app wanting
    attention;
  - `Meter`, an optional value 0–1 shown as a small bar;
  - `Graph`, up to 64 recent values 0–1 drawn as a sparkline, for system
    monitors;
  - `Menu`, an object path implementing `Menu1`.
- Methods:
  - `Activate(x, y)`: a primary click. Without `Activate`, a click opens
    the menu.
  - `SecondaryActivate(x, y)`: a middle click.
  - `Scroll(delta, orientation)`.
- **Zones**: the panel has left, center and right zones. The **panel
  layout** decides which items appear where, and in which order. It is
  an ordered list of item ids per zone, kept in GSettings
  (`org.helix.shell` `panel-left`, `panel-center`, `panel-right`;
  `contracts/gsettings/org.helix.shell.md`). Built-in items have ids
  too: `helix:apps`, `helix:clock`, `helix:status`, `helix:power` and
  others (`contracts/wayland/README.md`, "Panel layout"). So users arrange everything in one place, and can
  remove built-ins they do not want.
- A newly enabled extension's panel items are added at the end of the
  first zone they suit. The user moves them with the panel's "Customize
  Panel…" dialog (or by dragging in the panel's edit mode, a later
  phase).

### Indicators (StatusNotifierItem)

Apps' tray icons use the existing freedesktop **StatusNotifierItem**
protocol (with `com.canonical.dbusmenu` menus). This is what Plasma,
Xfce and Ubuntu's AppIndicator extension show. Helix hosts it directly
(as `org.kde.StatusNotifierWatcher`), as a built-in panel item
`helix:tray`. No extension is needed: apps that already have a tray
icon just work. That alone covers the most installed GNOME extension
(AppIndicator support). Its contract is written separately; see Phase 0.

Extensions do not use StatusNotifierItem; their panel items are the
richer, checked equivalent.

### Quick settings (`org.helix.Extension1.Toggle`, `org.helix.Extension1.Slider`)

- **Toggle**: a switch in the quick-settings grid, next to Wi-Fi and
  Night Light.
  - Properties: `Title`, `Subtitle` (e.g. "30 min left"), `Icon`,
    `Active`, `Sensitive`, `Details` (optional; a `Menu1` path).
  - With `Details`, the toggle gets the "›" that opens its list under
    the switches, like Wi-Fi's networks.
  - Method: `SetActive(b)`. The shell shows the new state only when
    the extension's `Active` changes, so a refused change does not
    show as on.
- **Slider**: a row under Volume and Brightness.
  - Properties: `Title`, `Icon`, `Value` (0–1), `Sensitive`.
  - Method: `SetValue(d)`. While dragging, at most one call per frame
    is sent.
- Which toggles and sliders appear, and in which order, is part of the
  user's quick-settings layout (`org.helix.shell` `quick-settings`),
  which includes the built-in ones (`helix:wifi`,
  `helix:night-light`…).

### Dock items (`org.helix.Extension1.DockItem`)

Dock items are entries in the dock's places section, next to drives and
the Trash: a folder stack, a running service, a "now playing" item.

- Properties: `Icon`, `Label`, `Running` (the dot), `Count` (a badge),
  `Progress`, `Urgent`, `Menu`.
- Methods: `Activate()`, and `Drop(uris)` when the manifest declares
  `accepts-drops = ["text/uri-list"]`.
- Apps' own dock badges stay with `com.canonical.Unity.LauncherEntry`,
  as today.

### Menus (`org.helix.Extension1.Menu`)

A menu is one property, `Layout`: a tree of items (in the contract, a flat list where each entry names its parent, since D-Bus types cannot nest without bound). Each item has an id,
a kind (normal, check, radio, separator, section heading or submenu),
a label, an optional icon, `enabled` and `checked`. There is one method,
`Activated(id)`. Because the whole tree is a single property, the shell
gets every change as one update: there is no item-by-item protocol to
get out of step, unlike dbusmenu. The shell builds, positions and
navigates the menu itself.

### Commands and keybindings

Manifest `contributes.command` entries become named commands
(`org.example.Caffeine.toggle`):

- They are listed in the run dialog and in Settings' keyboard
  shortcuts.
- A suggested keybinding is used only if it is free. Otherwise the
  command is listed unbound, and the user is told.
- Running a command calls the extension's
  `org.helix.Extension1.Commands.Run(id)`, starting the extension first
  if needed (`onCommand` activation).

Extensions never grab keys themselves.

### Settings

The manifest's `[settings]` table declares typed keys:

- **Types**: `bool`, `int`, `double`, `string`, `enum` (with
  `choices`) and `string-list`.
- **Each key has**: `default`, an optional `min`/`max`, a `label` and
  optional `help`.
- **Storage**: the host keeps the values (in a file per extension,
  `$XDG_CONFIG_HOME/helix/extensions/<id>.toml`: GSettings would need a
  compiled schema per extension). The extension reads and writes them
  through `Host1.GetSetting`, `SetSetting` and the `SettingChanged`
  signal.
- **Settings page**: the Settings app's page for the extension is
  generated from the declaration. It works without the extension
  running, so there is no preferences code to break.
- **Extension-drawn windows**: an extension that needs a richer page can
  declare `settings-exec`, a program that opens its own window. It is
  an ordinary app.

### Popovers (later phase)

Some extensions need their own content: a clipboard history with
search, a calendar, a media player with artwork. For these, a panel
item or dock item may declare `popover = true` and the `popover`
permission. When the item is activated, the shell asks the extension
to show a window through a private Wayland protocol
(`helix_extension_popover_v1`). The shell places that window under the
item, as it places its own menus, and closes it on Escape or a click
outside. The extension draws it with any toolkit. It is an ordinary
Wayland client surface in the extension's own process: it cannot see
or draw over anything else, and a crash only closes the popover.

## The host interface (`org.helix.Extension1.Host`)

Served by the shell at `/org/helix/Extension1/Host` on each extension's
connection:

- `Hello(api_version) → (granted_api, permissions, info)`: the first
  call. It agrees on the API minor version in use, says which permissions
  are granted, and gives session facts (the Helix version, the locale).
- `GetSetting(key) → v`, `SetSetting(key, v)`, signal
  `SettingChanged(key, v)`.
- `Log(level, message)`: goes to the journal under the extension's id,
  and to the extension's page in Settings (its last errors).
- `ShowNotice(title, body, icon)`: a notification under the extension's
  name (permission `notifications`). Extensions may also use
  `org.freedesktop.Notifications` like any app.
- `InhibitIdle(reason) → handle`, `Uninhibit(handle)` (permission
  `inhibit-idle`).
- `Launch(desktop_id, uris)`: start an app (permission `launch-apps`).
- `RequestActivation()`: asks the shell to open the extension's popover
  or details list (for example from a keybinding).

Everything else (the network, files, other D-Bus services) an extension
does directly, like any program, within its sandbox (see below).

## Activation

Extensions start lazily, in the VS Code manner:

- `onStartup`: when the session starts (after the shell is up).
- `onVisible`: when one of its items first becomes visible (the panel
  is shown, the quick settings open). The shell draws the item from
  its manifest defaults until the extension reports.
- `onCommand:<id>`: when one of its commands runs.
- `onDBusName:<name>`: when a name appears on the session bus (for
  example, start a "now playing" item when an MPRIS player appears).

Before starting an extension, the host checks that its manifest is
valid and its API version is supported.

## Isolation and robustness

- **Own process, own scope**: each extension runs in a systemd user
  scope (`app-helix\x2dextension-<id>.scope`). This gives it a memory
  limit (default 256 MiB) and a CPU weight below the shell's, it can be
  stopped as a unit, and its resource use shows in system monitors
  under its name.
- **Never blocking**:
  - The shell makes every call to an extension asynchronously, with a
    deadline (250 ms for UI actions).
  - A late answer is ignored, and the item shows that it is busy.
  - Property reads happen off the drawing path; the shell draws the
    last good state.
- **Crashes**:
  - When an extension exits unexpectedly, its items show their manifest
    defaults (dimmed), and the host restarts it after 1, 5, then 30
    seconds.
  - After 5 crashes in 10 minutes, the host stops trying for the
    session. It shows one notice ("Caffeine stopped working"), and the
    extension's page shows its last log lines.
- **Hangs**: an extension that misses 3 deadlines in a row is treated
  as crashed: its scope is stopped and the crash rules apply.
- **Checked input**: the content limits above, D-Bus type checking,
  ids checked against the manifest. An extension cannot add items its
  manifest does not declare.
- **Not on the lock screen**: no extension content is shown while the
  session is locked, and no extension command runs then.

## Permissions and sandboxing

Permissions are declared in the manifest and shown when the extension
is installed or enabled ("Caffeine can: keep the screen on").

- **API permissions** (enforced by the shell, from API 1): `notifications`,
  `inhibit-idle`, `launch-apps`, `popover`, `clipboard` (read the
  clipboard through the host).
- **System access** (enforced when sandboxed): `network`,
  `files:home`, `files:<xdg-dir>` (for example `files:downloads`),
  `session-bus` (talk to other session services), `system-bus:<name>`.
- **Sandbox** (later phase): extensions that declare `sandbox = true`
  run under bubblewrap with only the system access they declare,
  like a Flatpak app. Sandboxing becomes the default in API 2 once the
  common needs are covered; unsandboxed extensions then need the user
  to allow "full access", shown as such.
- **Updates**: an update that adds permissions stays disabled until
  the user accepts them.

In the first phases an unsandboxed extension can do whatever the user
can, like any program the user installs. The proposal is honest about
that. Process isolation protects the **desktop** from bugs from the
start; it does not yet protect the **user** from a malicious extension.

## Installation and enabling

- **Locations**:
  - system: `$XDG_DATA_DIRS/helix/extensions/<id>/`, from distribution
    packages; the `helix-apps` and `helix-desktop-full` packages can
    ship extensions this way;
  - per user: `$XDG_DATA_HOME/helix/extensions/<id>/`.

  A per-user copy hides a system one.
- **Enabled set**: `org.helix.extensions` `enabled` (ids), plus
  `disabled` for system extensions that are on by default
  (`enabled-by-default = true` in their manifest).
  `org.helix.extensions` `disable-all` turns every extension off, for
  troubleshooting. (Holding Shift while logging in to do the same for
  one session is left for later.)
- **Tools**: a `helix-extensions` command (list, info, enable, disable,
  install from an archive or directory, remove, logs), and an
  Extensions page in Settings that lists them, toggles them, shows
  their permissions, and opens their settings and logs.
- **No store in API 1**: extensions come from distributions, from
  archives, or from their authors' sites. A registry with review and
  signing is a later proposal.

## Versioning and compatibility

- **API version `MAJOR.MINOR`**:
  - Interface names carry the major version (`org.helix.Extension1.*`).
  - Minor versions only add: new properties, methods, contribution
    points or limits raised.
  - An extension asks for the minimum minor version it needs (`api`);
    `Hello` confirms it.
- **Within a major**, nothing is removed or changes meaning. A
  behavior change that could break an extension needs a new major.
- **Old majors**: Helix supports the previous major for at least two
  releases after a new one, serving both. Deprecations are announced in
  the spec one release ahead and logged when used.
- **Contracts**: each interface is a contract in `contracts/extensions/`
  (XML plus behavior notes, like `contracts/dbus/`). The implementation
  is tested against it with the existing conformance tests. Example
  extensions in the implementation repository double as integration
  tests in a nested session.

## Built-ins use the same model

The shell's own panel items, toggles and dock places get ids and go
through the same layout settings and drawing as extension items
(though they run in the shell). This proves the slots are enough for
real features, keeps one way of arranging things, and keeps extension
items from looking second-class.

## Developer experience

- **SDK**: a Rust crate, `helix-extension`, with typed items (`PanelItem`,
  `Toggle`, `Menu`), the `Hello` handshake, settings and logging, built
  on zbus. A Python module is a likely second.
- **Development mode**: `helix-extensions run ./my-extension` starts a
  directory without installing it, and restarts it when its files
  change. Its log is printed to the terminal.
- **Examples**, also used as tests:
  - a counter panel item;
  - a caffeine toggle;
  - a CPU graph;
  - an MPRIS "now playing" dock item;
  - a clipboard history (popover).
- **Documentation**: the contracts are the reference. A guide covers
  the manifest, the lifecycle and the limits.

## Mapping popular GNOME extensions

| GNOME extension | In Helix |
|---|---|
| AppIndicator / KStatusNotifierItem support | Built in (StatusNotifierItem host) |
| Dash to Dock / Ubuntu Dock | Built in (the dock) |
| Caffeine | Extension: toggle + `inhibit-idle` |
| Vitals / system monitors | Extension: panel item with `Graph` / `Meter` |
| Clipboard Indicator | Extension: panel item + popover + `clipboard` |
| Weather / OpenWeather | Built-in weather, or an extension: panel item + popover |
| GSConnect | Extension: panel item, toggle, dock item; its windows are its own app |
| Blur my Shell, Just Perfection (restyling) | Not extensions: Helix settings where wanted |
| Tiling assistants | Built in (`contracts/wayland/README.md`, "Tiling") |

## Phases

0. **Prerequisites**:
   - a StatusNotifierItem host;
   - panel and quick-settings layout settings, with ids for the
     built-ins;
   - the layout editor. Helix has no Settings app of its own, so this
     is the shell's "Customize Panel…" dialog.

   This is useful on its own. Its contracts: `org.helix.shell`, "Panel
   layout" and `org.kde.StatusNotifierItem.md`.
1. **API 1.0**:
   - the extension host;
   - the manifest, activation, the `Host` interface;
   - panel items, menus, toggles, sliders, settings;
   - API permissions, crash handling, scopes;
   - `helix-extensions`, the Settings page;
   - the Rust SDK and examples.
2. **API 1.x**: dock items, commands and keybindings, popovers,
   `onDBusName` activation, panel edit mode by dragging.
3. **Sandboxing and distribution**: bubblewrap sandbox and system
   permissions, then a registry with signing (separate proposal).

## Alternatives considered

- **In-process plugins (shared libraries)**, the Xfce/Plasma model.
  - For: fast, and they can draw anything.
  - Against: a crash or hang takes the shell down. Rust has no stable
    ABI, and a C ABI would freeze the shell's internals the same way.
  - Rejected for the isolation goal.
- **An embedded script engine** (JavaScript or Lua) inside the shell,
  the GNOME model.
  - Against: same process, same failures. Scripts drift toward
    patching internals unless the engine is locked down to a fixed API,
    which ends up being this design plus an engine.
- **WebAssembly components in the shell** (wasmtime).
  - For: strong sandboxing, small, a typed interface (WIT).
  - Against: extensions need D-Bus, files and the network, which means
    a large host-provided system interface. It limits languages and
    libraries, and a busy component still shares the shell's threads
    unless run elsewhere.
  - Worth revisiting for tiny, pure widgets; not as the base.
- **A Wayland protocol for everything** (each item a surface).
  - For: maximal freedom.
  - Against: items drawn by extensions would look inconsistent, ignore
    the theme and scale, be inaccessible, and cost a client surface
    each. A state-and-event API fits panel items far better. Surfaces
    are kept for popovers only.
- **The session bus instead of a private connection**.
  - For: simpler tooling.
  - Against: name clashes, and anything on the bus could pose as an
    extension or as the shell. A private connection per extension
    identifies both ends without checks. The host can still expose a
    debugging view.

## Open questions

- Panel item **width**: fixed by content (label and icon), or may an
  extension ask for a width (a graph)? Proposed: content-sized, with a
  `Graph` fixed at a standard width.
- **Several instances** of one item (two clocks for two time zones),
  as Plasma allows: needed in API 1, or later? It would make the item
  id `<id>#<n>`, with per-instance settings.
- **Multi-monitor**: is an item on every monitor's panel? Is there one
  process for all? Proposed: one process; items appear on every panel
  showing that zone, and calls carry the monitor's connector.
- **Translations**: gettext catalogs in the extension directory, with
  the manifest's text looked up by key? Or localized manifest fields
  (`name.de = …`)? Proposed: localized fields, as `.desktop` files do.
- Should the shell offer **shared data** extensions often fetch
  themselves (battery, network, now playing), to avoid each extension
  polling the same services?
- Exact **limits and deadlines**: the numbers above are starting points
  and should be measured.
