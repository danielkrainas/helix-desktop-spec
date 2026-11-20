# Agent Access, version 1

A Helix extension with no GNOME counterpart (helixdesktop/spec#1). It
lets an agent (a coding assistant, a background automation) read the
state of the user's session and act on it, behind a permission the user
controls in **Settings › Privacy & Security › Agent Access**.

| File | What |
|------|------|
| `cli.md` | `helix-agent`, the command-line front end: commands, `--help`, JSON output, exit codes and messages |
| `org.helixdesktop.Agent1.xml` | what agents call; served by the shell |
| `org.helixdesktop.AgentAccess1.xml` | what the Settings page calls; served by the shell |
| `../gsettings/org.helixdesktop.agent-access.md` | the settings (schema `org.helixdesktop.agent-access`) |

Both interfaces are on the shell's main connection (bus name
`org.gnome.Shell`, not the connection sandboxed apps commonly reach;
`contracts/dbus/README.md`, "The shell's two connections"), at
`/org/helixdesktop/Agent1`.

**The shell enforces the policy, not the CLI.** `helix-agent` is a thin
client with no policy of its own; a program that talks to
`org.helixdesktop.Agent1` directly meets the same checks. Agent Access is
off by default.

## What it protects against, and what not

Agent Access decides what a program may do *through the desktop*: read
windows and notifications, launch apps, drive apps through the
accessibility tree, take screenshots and send input. It gives the user a
switch, a consent prompt, a visible indicator, exclusions and a log.

It is not isolation from programs the user runs unconfined. Such a
program runs as the user and can already change the user's settings
(including these keys) and files, or start other programs. Agent Access
makes sure that **no D-Bus call lets an agent give itself access**: the
only way to a grant is the user answering the shell's consent prompt,
which an agent can neither answer over D-Bus nor through synthetic input
(helixdesktop/desktop#143). Changing the settings never grants anything
by itself (see "Grants"). Sandboxed apps (Flatpak, snap) are refused
outright.

## Access levels

| Level | Value in `access-level` | Allows |
|-------|-------------------------|--------|
| Read only | `read-only` | reading state (windows, apps, workspaces, monitors, notifications, recent files, file search), waiting for windows, and talking to the user (ask, confirm, remind) |
| Control the desktop | `desktop` | also launching apps, opening files and URLs, managing windows and workspaces, quick settings toggles, volume and brightness, notifications from the agent, setting the clipboard |
| Control apps | `apps` | also reading and acting on apps' accessibility trees |
| Screen and input | `screen-and-input` | also screenshots and synthetic pointer and keyboard input |

Each level includes the ones above it. Notification contents and the
clipboard need their own switches (`notifications`, `clipboard`) on top
of the level.

## Methods

Every method of `org.helixdesktop.Agent1` runs the policy check below
before doing anything. "Level" is the level the method needs;
"Sensitive" says when it counts as a sensitive action (see "Sensitive
actions"). Every method with an `options` argument accepts the common
options `interactive` (`b`, default true: false means the shell never
asks the user during this call; where it would have to, the call fails
with `NotGranted` or `NeedsConfirmation`) and `sensitive` (`b`, default
false: the agent marks this call sensitive).

| Method | Level | Sensitive | Notes |
|--------|-------|-----------|-------|
| `GetStatus` | none | | never asks the user; answers when access is off |
| `RequestAccess` | the one asked for | | |
| `GetState` | read-only | | |
| `ListWindows`, `ListApps`, `ListWorkspaces`, `ListMonitors` | read-only | | |
| `ListNotifications` | read-only | | contents only with `notifications` |
| `RecentFiles`, `SearchFiles` | read-only | | |
| `WaitForWindow`, `WaitForWindowClosed` | read-only | | |
| `Ask`, `Confirm` | read-only | | |
| `AddReminder`, `ListReminders`, `CancelReminder` | read-only | | an agent sees only its own reminders |
| `GetClipboard` | read-only | | needs `clipboard` |
| `Open`, `Launch` | desktop | when marked | not excluded apps (also as the app that would open a file) |
| `WindowAction` | desktop | `close`, or when marked | not excluded apps' windows |
| `SwitchWorkspace` | desktop | when marked | |
| `SetToggle` | desktop | `wifi`, `airplane-mode`, or when marked | |
| `SetValue` | desktop | when marked | |
| `Notify` | desktop | when marked | labelled with the agent; at most 5 a minute |
| `SetClipboard` | desktop | when marked | needs `clipboard` |
| `A11yTree` | apps | | |
| `A11yAction`, `A11ySetValue` | apps | when marked | |
| `Screenshot` | screen-and-input | | |
| `Input` | screen-and-input | when marked | |

The shell implements the methods in stages (helixdesktop/desktop#146
to #151); until a method is implemented it is absent from introspection
and answers `org.freedesktop.DBus.Error.UnknownMethod`. `GetStatus` and
`RequestAccess` are always there.

`GetStatus()` and `RequestAccess(level, options)` return the caller's
status (`a{sv}`):

| Key | Type | Meaning |
|-----|------|---------|
| `version` | `u` | Agent Access version (1) |
| `enabled` | `b` | the master switch |
| `access-level` | `s` | the configured level (the most a grant can be) |
| `granted-level` | `s` | the level this agent may use now (the lower of its grant and `access-level`), `""` without a grant |
| `grant` | `s` | `none`, `session`, `always` or `denied` |
| `agent` | `s` | the agent's id (see "Who is calling") |
| `executable` | `s` | the agent's executable |
| `name` | `s` | the self-declared display name, `""` if none |
| `identified` | `b` | the agent could be told apart from the CLI (see "Who is calling") |
| `remote` | `b` | the caller is in a remote session |
| `allow-remote` | `b` | the `allow-remote` switch |
| `locked` | `b` | the screen is locked |
| `notifications`, `clipboard`, `confirm-sensitive` | `b` | the switches of those names |

`RequestAccess` asks for a grant of `level` (`""`: the configured
`access-level`), going through the same checks as any method of that
level; an agent that needs only reading can ask for `read-only` and the
consent prompt asks for no more. It returns the status after the grant.

`AccessChanged` is emitted whenever something `GetStatus` reports may
have changed (a setting, a grant, the lock); it carries nothing, so it
tells no agent about another.

### Records

Methods returning records use dictionaries with these keys (more may be
added; clients ignore keys they do not know):

- **Window**: `id` (`u`, the id of the compositor's window list, also
  `org.helixdesktop.ScreenCastChooser`'s), `app-id` (`s`), `title`
  (`s`), `workspace` (`u`, index from 0; for a window on every
  workspace, the active one), `all-workspaces` (`b`, on every
  workspace), `monitor` (`s`, connector of the monitor showing most of
  it), `focused` (`b`), `x`, `y`, `width`, `height` (`i`, logical
  pixels), `state` (`s`: `normal`, `maximized`, `fullscreen`,
  `minimized`, `tiled-left`, `tiled-right`), `document` (`s`, URI of
  the open document, only when the app tells it; see "Open
  documents").
- **App**: `id` (`s`, the desktop file's name without `.desktop`, or
  the window's app id for windows without a desktop file), `name` (`s`),
  `running` (`b`), `windows` (`u`).
- **Workspace**: `index` (`u`), `active` (`b`), `windows` (`u`; windows
  on every workspace count on each).
- **Monitor**: `connector` (`s`), `name` (`s`), `x`, `y`, `width`,
  `height` (`i`, logical), `scale` (`d`), `primary` (`b`).
- **State** (`GetState`): `focused-window` (`u`, 0 for none),
  `active-workspace` (`u`), `dark-style`, `do-not-disturb`,
  `night-light`, `wifi`, `airplane-mode`, `muted` (`b`), `volume` (`d`,
  0–1, above 1 when over-amplified), `brightness` (`d`, 0–1). `wifi`
  is absent without NetworkManager, `airplane-mode` without
  NetworkManager and Bluetooth, `volume` and `muted` without a sound
  server, `brightness` without a controllable screen. `focused-window`
  is 0 when the focused window is an excluded app's.
- **Notification**: `id` (`u`), `app-id` (`s`), `app-name` (`s`),
  `time` (`x`, Unix seconds), and with `notifications` on, `summary` and
  `body` (`s`); without it, `redacted` (`b`, true).
- **File**: `uri` (`s`), `name` (`s`), `mime-type` (`s`), `modified`
  (`x`, last changed; for a recent item that is not a local file, when it
  was used), `size` (`t`, local files only), and from `RecentFiles`
  `used` (`x`, when it was last used).
- **Reminder**: `id` (`s`), `time` (`x`), `text` (`s`).
- **Accessible node**: `id` (`s`), `role` (`s`, AT-SPI role name),
  `name`, `description` (`s`), `states` (`as`), `value` (`s` or `d`;
  never for password fields), `actions` (`as`), `children` (`aa{sv}`).
- **Image** (`Screenshot`): `png` (`ay`), `width`, `height` (`i`,
  pixels), `scale` (`d`).
- **Clipboard** (`GetClipboard`): `mime-type` (`s`), and `text` (`s`)
  for text types or `data` (`ay`) otherwise.

The methods' own options and arguments are specified with their
implementation (helixdesktop/desktop#146–#151) and listed in `cli.md`
through the commands that use them.

### Read only methods

The options and arguments of the *Read only* methods
(helixdesktop/desktop#146), besides the common ones. Malformed values
fail with `InvalidArgs`.

| Method | Options and arguments | Returns |
|--------|----------------------|---------|
| `GetState` | | the State record |
| `ListWindows` | `app` (`s`), `title` (`s`, contained in the title, ignoring case), `focused` (`b`) | Window records, most recently used first |
| `ListApps` | `running` (`b`: only apps with windows) | App records, by name: the apps the app grid shows, and apps with windows |
| `ListWorkspaces` | | Workspace records, by index |
| `ListMonitors` | | Monitor records |
| `ListNotifications` | `limit` (`u`, default and most 50) | Notification records of the notification list, newest first |
| `RecentFiles` | `limit` (`u`, 1–200, default 50), `type` (`s`, a MIME type or `major/*`) | File records, most recently used first |
| `SearchFiles` | `query` (not empty); `since`, `until` (`x`, Unix seconds, compared with `modified`), `type` (`s`), `in` (`s`, an absolute folder path or `file://` URI, default the home folder; `NotFound` when it is no folder), `limit` (`u`, 1–200, default 50) | File records |
| `WaitForWindow` | `match` holds `app`, `title`, `focused` as for `ListWindows` (and the common options); `timeout_ms` (0: no limit) | the Window record of the first matching window, at once when one is open; `TimedOut` |
| `WaitForWindowClosed` | `window`; `timeout_ms` (0: no limit) | when the window is gone; `NotFound` when it is not open at the start, `TimedOut` |

- **Exclusions.** Windows of excluded apps (Settings and `excluded-apps`)
  are never listed, counted, matched or returned; `ListWindows` or
  `WaitForWindow` with `app` naming an excluded app, and
  `WaitForWindowClosed` of an excluded app's window, fail with
  `Excluded`. Notifications are left out when their app id is excluded,
  when they are about an excluded app's window (the shell's "… is
  ready"), or, without an app id, when their app name is an excluded
  app's name. Recent items are left out when marked private, or when
  every app that registered them is excluded (compared by the app's id,
  name, `StartupWMClass` and executable).
- **Notifications** without the `notifications` switch have no `summary`
  and `body`, and `redacted` true. Bodies with markup are given as plain
  text.
- **Recent files** come from `$XDG_DATA_HOME/recently-used.xbel`; none
  when `org.gnome.desktop.privacy` `remember-recent-files` is off, and
  none older than `recent-files-max-age`. Local files that no longer
  exist are left out; other locations (`sftp://`…) are kept, without
  `size`.
- **File search** asks LocalSearch (`org.freedesktop.LocalSearch3`, else
  `org.freedesktop.Tracker3.Miner.Files`; neither is started for it) for
  files below the folder whose names or contents have words starting
  with the query's words; when neither runs or it finds nothing, the
  shell looks through the folder itself (names containing the query,
  then text files of at most 256 KiB containing it; at most 20,000
  entries looked at). Hidden files and folders are left out. The filters
  apply to what was found; results are by relevance (names before
  contents).
- **Waits** look at the windows again a few times a second; while one
  runs the agent counts as active (the indicator shows), and when the
  agent loses access meanwhile (a setting, a revoke, the lock) it ends
  with the error a new call would get.

### Open documents

*Helix's choice:* a Window record's `document` comes from what apps
already tell the desktop, so no app needs changes, and never from the
app's accessibility tree (that is the *Control apps* level's):

1. A media player's MPRIS `Metadata` `xesam:url`, when its `DesktopEntry`
   names the window's app: for that app's most recently used window.
2. Otherwise the item of `recently-used.xbel` that the window's app
   registered (by app id, name or executable) and used most recently,
   whose file name (at least 3 characters) is in the window's title:
   Text Editor, Document Viewer, Image Viewer and other apps that record
   their recent files and show the file name in the title.

Apps that do neither have no `document`.

### Control the desktop methods

The options and arguments of the *Control the desktop* methods
(helixdesktop/desktop#147), besides the common ones. Malformed values
fail with `InvalidArgs`; none returns anything but `Notify`.

| Method | Options and arguments | Does |
|--------|----------------------|------|
| `Open` | `uri`: an absolute path, a `file://` URI, or an `http`, `https` or `mailto` address; `app` (`s`, an app id: open it with that app) | opens it in its default app (see below); `NotFound` when the file does not exist or no app opens it |
| `Launch` | `app_id` (with or without `.desktop`); `uris` as `Open`'s `uri` | starts an app the app grid shows, handing it `uris`; `NotFound` for other ids |
| `WindowAction` | `window`; `action`: `focus`, `close`, `minimize`, `maximize`, `unmaximize`, `restore`, `fullscreen`, `unfullscreen`, `tile-left`, `tile-right`, `move` (options `x`, `y`, `i`), `resize` (`width`, `height`, `i`, at least 1), `to-workspace` (`workspace`, `u`) | acts on the window; `NotFound` for an unknown window or workspace, `NotSupported` for an unknown action |
| `SwitchWorkspace` | `index` (from 0) | makes it the active workspace; `NotFound` when there is no such workspace |
| `SetToggle` | `name`: `dark-style`, `do-not-disturb`, `night-light`, `wifi`, `airplane-mode`, `mute`; `on` | turns it on or off; `NotSupported` for other names and for what is not there (no NetworkManager, sound server…) |
| `SetValue` | `name`: `volume`, `brightness`; `value` (`d`, 0–1 as the State record has it; `volume` up to 1.5 with over-amplification on) | sets it; `NotSupported` as for `SetToggle` |
| `Notify` | `title` (not empty), `body`; `icon` (`s`, a theme icon's name), `urgency` (`s`: `low` or `normal`, default `normal`) | shows a notification; returns its id (`u`) |

- **Through the shell's own ways.** *Helix's choice:* the methods do
  what the user's own clicks do, in the same code: apps start through
  the launcher (in their own scope, with an activation token), files
  open as a double-click on the desktop opens them, windows are acted on
  through the compositor's window list as the window menu and the
  overview do, and toggles and values change as quick settings changes
  them (setting the volume unmutes, as its slider does). Nothing an
  agent does through these methods is out of the user's reach to undo.
- **Opening.** `Open` hands a local file or folder to the default app
  for its type, and an address to the default app for its scheme.
  Other schemes (`ssh:`, `smb:`, apps' own schemes) are refused with
  `NotSupported`: an app's own scheme can start things without asking.
  A `.desktop` file launches its application only when it is trusted as
  on the desktop (an application's entry, executable, not writable by
  others, and marked `metadata::trusted` with *Allow Launching*);
  otherwise it is a plain file and opens as one. Relative paths are
  refused; `helix-agent` makes them absolute.
- **Exclusions.** The app that would start is checked like a target
  (step 9): the `app` option, `Launch`'s `app_id`, the default app for
  the file or address, or the trusted `.desktop` file's name. Excluded
  apps, Settings, and any app whose command runs Settings' program
  (`helix-settings-app`) are refused with `Excluded`, before the user is
  asked about a sensitive action. `WindowAction` on a window of an
  excluded app (or of Settings) fails with `Excluded`.
- **Windows.** `focus` raises the window and gives it the keyboard,
  unminimizing it and switching to its workspace; `restore` makes it
  normal again (unminimized, and neither maximized, tiled nor full
  screen); actions that are already so do nothing (`tile-left` on a
  window tiled left does not untile it). `move` places the window's own
  area (`x`, `y` of its Window record) and `resize` asks its app for a
  size, both ending a maximize, tile or full screen first, and fail
  (`Failed`) for a minimized window or one on another workspace. The
  compositor keeps a moved window on a monitor: its top below the top
  bar and at least 64 pixels of it on the monitor. *Helix's choice:*
  the call returns once the windows show the change, or after at most
  two seconds (an app may choose another size, or ask before closing),
  so a `ListWindows` right after it shows the result.
- **Quick settings.** `wifi` is NetworkManager's Wi-Fi switch and
  `airplane-mode` turns every radio off (or back on), as quick settings
  does; both are sensitive, since they change the network. With
  `confirm-sensitive` on, the sensitive-action prompt names what is
  changed ("turn Wi-Fi off"), as it does for closing a window.
- **Notifications** appear under "Agent: <agent>" in place of an app's
  name, `<agent>` being the agent's display name and its executable's
  file name (or the executable and the script) as the indicator shows
  it, so an agent cannot pass one off as the system's or an app's. They
  are plain text (no markup, no buttons, never critical: Do Not Disturb
  holds them back like any app's) and have no app id. *Helix's choice:*
  an agent may show at most 5 in any minute; more fail with `Failed`
  until the minute is up.

## Errors

The shell answers refusals with these D-Bus errors (prefix
`org.helixdesktop.Agent1.Error.`). The message is English for logs;
clients key on the name. Where the message is listed it is exactly
that token, for clients to word their own message.

| Error | When | Message |
|-------|------|---------|
| `Sandboxed` | the caller is in an app sandbox, or cannot be identified as an unsandboxed process of the session's user | |
| `AccessOff` | `enabled` is off | |
| `Remote` | the caller is in a remote session and `allow-remote` is off | |
| `Locked` | the screen is locked (or locking) | |
| `LevelTooLow` | the method needs a level above `access-level` | the level needed |
| `FeatureOff` | the method needs a switch that is off | `clipboard` |
| `NotGranted` | no grant, and the user could not be asked (`interactive` false, no consent prompt available, or the prompt went away unanswered) | |
| `DeniedByUser` | the user denied access, or declined a sensitive action | |
| `Excluded` | the target is excluded | `app:<app id>`, `settings`, `password-field`, `system-prompt` or `shell` |
| `NeedsConfirmation` | a sensitive action with `confirm-sensitive` on, and `interactive` false | |
| `Cancelled` | the user dismissed `Ask` or `Confirm` | |
| `TimedOut` | a wait or question ran out of time | |
| `NotFound` | no such window, app, node or reminder | |
| `NotSupported` | a name or action this shell does not support (e.g. a toggle without hardware) | |
| `Failed` | anything else | |

Arguments that are malformed fail with
`org.freedesktop.DBus.Error.InvalidArgs` (after the policy check, so an
agent without access learns nothing from them).

## The policy check

Each call is checked in this order; the first that fails decides the
error:

1. **Sandbox.** `require_unsandboxed`: refused (`Sandboxed`) when the
   caller is in a Flatpak or snap, or has another root directory, or its
   credentials are missing or another user's.
2. **Identity.** The agent is identified (see "Who is calling").
3. **Master switch.** `enabled` off: `AccessOff`.
4. **Remote.** In a remote session with `allow-remote` off: `Remote`.
5. **Lock.** Screen locked or locking: `Locked`.
6. **Level.** The method's level above `access-level`: `LevelTooLow`.
7. **Switches.** `clipboard` off for the clipboard methods:
   `FeatureOff`.
8. **Grant.** Denied this session: `DeniedByUser`. A grant covering the
   level: go on. Otherwise ask the user (see "Consent"), unless
   `interactive` is false (`NotGranted`); after the answer, steps 3–6
   are checked again.
9. **Exclusions.** The target (window, app, node, the screen) is checked
   against the exclusions: `Excluded`.
10. **Sensitive actions.** Sensitive, with `confirm-sensitive` on: ask
    the user (`NeedsConfirmation` when `interactive` is false;
    `DeniedByUser` when declined).

`GetStatus` runs steps 1 and 2 only. The outcome of every call is
recorded in the activity log (see "Activity log").

## Who is calling

The shell asks the bus for the caller's credentials
(`GetConnectionCredentials`): its user (must be the shell's), its
process id and security label. From `/proc/<pid>` it reads the
executable (`exe`, a trailing ` (deleted)` removed), the parent process,
the command line and the environment.

An agent usually runs `helix-agent` through a shell, so the D-Bus caller
is the CLI, not the agent. *Helix's choice:* the shell walks up from the
caller through its parent processes, skipping processes whose
executable's file name is one of: `helix-agent`, `gdbus`, `busctl`,
`dbus-send`, `sh`, `dash`, `bash`, `zsh`, `fish`, `ksh`, `mksh`, `tcsh`,
`csh`, `env`, `timeout`, `nice`, `ionice`, `nohup`, `setsid`, `stdbuf`,
`xargs`, `flock`, `time`, `script`, `taskset`, `chrt`. The first process
that is not skipped is **the agent**. The walk stops (at most 32 steps)
at process 1, at a process of another user, or at the user's service
manager (`systemd`); if it stops before finding one, the agent is the
caller itself and is **unidentified**.

- The agent's **id** is its executable's path. For an interpreter
  (file name `python`, `python3`, `python3.<n>`, `pypy3`, `node`,
  `nodejs`, `bun`, `deno`, `perl`, `ruby`, `php`) the id is the
  executable, a space, and the script: the first command-line argument
  not starting with `-`, made absolute against the process's working
  directory (`python3 -m <module>`: `-m <module>`). An interpreter
  running inline code (`-c`, `-e`) or reading standard input has no
  script and is unidentified.
- An **unidentified** agent (also when run from a terminal by hand: the
  agent is then the terminal app) can be granted access for the session
  only; the consent prompt does not offer *Always allow* for it.
- **Display name:** the shell reads `HELIX_AGENT_NAME` from the
  environment of the caller, else of the agent: at most 64 characters,
  control characters removed. It is shown next to the executable in the
  consent prompt, the indicator, Settings and the log, and is **never
  used for identity**: two agents with the same name and different
  executables are different agents.

A program that changes its executable's path (e.g. every update
installs to a new directory) is a new agent and is asked again.

## Grants

A grant is (agent id, level, kind). Kinds:

- **session**: in the shell's memory, until the session ends, the user
  revokes it (Settings, or *Stop* in the indicator), or `enabled` is
  turned off;
- **always** (*Always allow*): remembered across sessions, honoured only
  while `remember-mode` is `remember`, until the user forgets it in
  Settings. *Stop* in the indicator suspends it for the rest of the
  session (see "Indicator").

The level of a grant is the level the user approved in the consent
prompt. An agent may use the lower of its grant's level and
`access-level`. Raising `access-level` (by anyone) widens no grant: an
agent that needs more is asked again, and the user approves the higher
level in the prompt. A denial (*Deny*) is remembered for the session:
the agent gets `DeniedByUser` without being asked again, until the user
revokes or forgets it in Settings or `enabled` is turned off.

Remembered grants are kept in `$XDG_CONFIG_HOME/helix/agent-grants.toml`
(mode 0600, written whole through a temporary file and a rename):

```toml
version = 1

[[agent]]
id = "/usr/bin/node /home/me/.local/lib/agent/cli.js"
executable = "/usr/bin/node"
name = "Example Agent"         # the last display name seen; for showing only
level = "desktop"
granted = 1795132800           # Unix seconds
```

The shell reads the file once when it starts and writes it whenever a
remembered grant changes; changes made to the file by others while the
shell runs are overwritten. Entries with an unknown `level` are ignored;
a newer `version` is read for the entries it understands and kept.

## Consent

When a call needs a grant the agent does not have, the shell asks the
user in a shell-modal prompt (helixdesktop/desktop#143) naming the
agent (display name and executable) and the level: *Deny*, *Allow for
this session*, and with `remember-mode` `remember` and an identified
agent, *Always allow*. The prompt asks for the level the call needs, or
`access-level` when that is higher (`RequestAccess` asks for exactly
its `level`). One prompt is shown at a time; other calls wait for it,
and a call from the same agent that the answer covers goes on without a
second prompt. The prompt cannot be answered over D-Bus or by an
agent's synthetic input; locking the screen dismisses it unanswered
(`NotGranted`).

*Helix's choice* for how the prompt resists the agent: the answer
travels only inside the shell (no D-Bus method answers it); the shell
refuses accessibility actions on the prompt (screen readers still read
it, and the keyboard works it); the prompt's layer surface has the
namespace `helix-agent-prompt`, and while one is shown the compositor
drops input that does not come from the machine's own devices (remote
desktop and EIS input, and the headless backend's test input; key,
button and touch releases still go through so nothing stays held); and
keys and clicks in the first second after the prompt appears are
ignored, so a prompt timed to appear under a click or an Enter meant for
something else is not answered by it. *Deny* has the keyboard focus at
first. A remote desktop user answers it at the machine. If another
dialog (a password prompt) needs the screen while the prompt is up, the
prompt waits and comes back after it.

## Indicator

While any agent holds a grant and has made an allowed call in the last
minute, or has a call running (a wait, a question, input), the top bar
shows an indicator, like the screen-sharing one (helixdesktop/desktop#143).
It is mandatory, not a setting; it is not shown while the screen is
locked. Its menu lists the active agents (display name and executable,
and the level) with *Stop* for each, and opens Settings at the Agent
Access page (`helix-settings-app agent-access`; GNOME's Settings, which
has no such page, at Privacy).

*Stop* ends that agent's session grant and, *Helix's choice*, also
suspends its remembered (*Always allow*) grant for the rest of the
session, so the agent's next call asks the user again instead of going
on (a *Stop* that an always-allowed agent could ignore would stop
nothing). Answering the prompt again lifts the suspension; Settings'
`Revoke` also does. Unlike *Deny*, *Stop* does not refuse the agent's
next request: it asks.

## Sensitive actions

A call is sensitive when the table in "Methods" says so, or when the
agent passes `sensitive` true (agents and the CLI mark deleting files,
sending messages or email, purchases and similar actions; the shell
cannot know what pressing a button in an app does). With
`confirm-sensitive` on, the shell asks the user every time, naming the
agent and the action, before doing it.

## Remote sessions

*Helix's choice:* a caller is remote when its process, or the agent's,
belongs to a logind session (`GetSessionByPID`, see
`contracts/dbus/org.freedesktop.login1.md`) whose `Remote` property is
true, or when its environment has `SSH_CONNECTION` or `SSH_CLIENT`.
Processes outside any session (`NoSessionForPID`, e.g. apps the shell
started in their own systemd scopes) count as local. When logind cannot
be asked at all, callers count as remote (so `allow-remote` is needed).
This guards against agents in SSH sessions by mistake; a program that
starts itself outside every session counts as local.

## Hard exclusions

Not settings; the user cannot turn them off:

- **The lock screen.** Nothing runs while the screen is locked (step 5).
- **Password fields.** Accessible nodes with the role `password text`
  or the state `PASSWORD` have no `value`; acting on them, setting them,
  or sending input while one has focus fails with `Excluded`
  (`password-field`). Screenshots black them out where the shell knows
  their bounds.
- **System prompts.** While the shell shows a polkit, keyring, network
  secret, Bluetooth pairing, Agent Access consent or sensitive-action
  prompt, `A11y*`, `Screenshot`, `Input` and the clipboard methods fail
  with `Excluded` (`system-prompt`); the shell's own UI is never a
  target (`shell`).
- **Settings.** *Helix's choice:* every window of the Settings app
  (`org.helixdesktop.Settings`) is excluded (`settings`), because the
  shell cannot tell which page a window shows and the Agent Access page
  must never be reachable; Settings also holds other security pages.
- **User exclusions:** the apps in `excluded-apps`
  (`app:<app id>`).

Excluded apps' windows are left out of `ListWindows`, `ListApps` and
`WaitForWindow`, their notifications out of `ListNotifications`, and
they are blacked out in screenshots; launching or opening them
(also opening a file whose default app is one), and any method naming
one of their windows, fails with `Excluded`. App ids are compared
without a `.desktop` suffix.

## Settings take effect at once

The shell reads the settings on every call and watches them: turning
`enabled` off ends every session grant and denial; lowering
`access-level` narrows every grant; turning a switch off refuses the
next call that needs it. Revoking an agent (Settings, or *Stop*)
ends its grant at once. A call already waiting (a wait, a question, a
prompt) when its agent loses access ends with the error the agent would
now get.

## Who may manage grants

`org.helixdesktop.AgentAccess1` is for the Settings page only. *Helix's
choice:* the shell accepts a call when the caller is unsandboxed, of the
session's user, and its executable is `helix-settings-app` in the
directory of the shell's own executable or in `/usr/bin`; any other
caller gets `org.freedesktop.DBus.Error.AccessDenied`. That interface
has no method that grants or widens access, so a program passing as
Settings could only take access away, list agents or clear the log;
grants come only from the consent prompt.

- `ListAgents()`: every agent with a grant or a denial this session, or
  a remembered grant: `agent` (`s`, id), `executable` (`s`), `name`
  (`s`), `level` (`s`), `kinds` (`as`: any of `session`, `always`,
  `denied`), `granted` (`x`, Unix seconds), `last-active` (`x`, Unix
  seconds of its last allowed call this session, 0 for none),
  `identified` (`b`).
- `Revoke(agent)`: ends its session grant and denial; a remembered grant
  stays (so under `remember` it is let in again).
- `Forget(agent)`: `Revoke`, and forgets its remembered grant.
- `StopAll()`: `Revoke` for every agent.
- `ClearLog()`: empties the activity log (helixdesktop/desktop#144) and
  records that it did (see "Activity log").
- `AgentsChanged` is emitted when the list changes.

Unknown agent ids are not an error (nothing to take away).

## Activity log

The shell records every `Agent1` call except `GetStatus`
(helixdesktop/desktop#144), refused calls included, and what the user
did about agents: consent-prompt and sensitive-action-prompt answers,
and *Stop* in the indicator. Settings reads the log directly (*View
Activity*) and clears it with `ClearLog`; `helix-agent log` reads it for
the user.

### Files

`$XDG_STATE_HOME/helix/agent-activity/` (directory 0700), one file per
local day, `YYYY-MM-DD.jsonl` (0600), named by the day of the entries'
`time`. Other files there are not the log's. Each line is one JSON
object followed by `\n`. Lines are appended, each in one write, so the
lines of concurrent calls never mix; a line torn by a crash is ended
by the next write (with a `\n` first), and readers skip lines that are
not a JSON object. Within a file, lines are in the order calls *ended*,
so `time` (when the call *arrived*) may go back a little; readers sort
by `time`. A call is recorded when it ends (a wait or a question that
the shell does not live to finish is not recorded); a refusal of the
policy check at once.

*Helix's choice:* the shell never follows a symlink there: the directory
must be a real directory of the user's (one that is not is not written
to, cleared or pruned), and the day files are opened with `O_NOFOLLOW`
(a symlink in their place is not written through). A directory or file
with a wider mode is narrowed to 0700 / 0600.

### Lines

Keys, in this order (readers must not depend on the order, and must
ignore keys they do not know; keys may be added):

| Key | Type | Value |
|-----|------|-------|
| `time` | integer | When the call arrived, Unix seconds. Always a number (readers may also take ISO 8601 text). |
| `agent` | string | The agent's id (see "Who is calling"); `""` when the caller could not be identified (e.g. sandboxed: then the caller's executable when it is the user's process, else `""`). |
| `name` | string | Its display name, `""` for none. |
| `method` | string | The `Agent1` method; or one of the *events* below. |
| `target` | string | What the call read or acted on, absent for none: an app id as the agent gave it (`org.gnome.Maps`), a window as `window:<id>` and, once known, ` (<app id>)` (`window:21 (org.gnome.TextEditor)`), a file or folder as a `file://` URI, an address as its scheme and host only (`https://example.org`, `mailto:`), a workspace as `workspace:<index>`, a quick setting or value by its name (`wifi`, `volume`); for Settings' `Revoke` and `Forget`, the agent's id. Never a window title. Control characters are removed and it is cut at 256 characters. |
| `level` | string | The access level the call needed (`read-only`, …); for a consent prompt the level asked, for *Stop* the level the agent held. Absent for calls needing none. |
| `outcome` | string | `allowed`, `refused:<Error>` or `cancelled` (below). |
| `error` | string | For an `allowed` call that then failed: the error's short name (`TimedOut`, `NotFound`, `NotSupported`, `Failed`, `InvalidArgs`). Absent when it succeeded. |
| `<kind>-length` | integer | The length in characters of something the agent sent, instead of it: `query-length` (`SearchFiles`), `title-length` (a window match's `title`, a notification's title), `body-length` (a notification's body); later stages add `text-length`, `question-length` and the like. |
| `uri-count` | integer | `Launch`: how many files or addresses it handed the app (absent for none). |
| `action` | string | `WindowAction`: the action (`tile-left`, `close`); a `SensitivePrompt`: what the prompt said the call does, where it said more than the method (`wifi-off`, `close`). |
| `state` | string | `SetToggle`: `on` or `off`. |
| `grant` | string | Consent prompts: `session` or `always`. |
| `count`, `until` | integer | Coalesced refusals (below): how many calls the line stands for (≥ 2), and the last one's `time`. |
| `dropped` | string | `LogFull` lines (below): `refusals` or `all`. |

Outcomes:

- `allowed`: the policy check let the call through. It may still have
  failed (`error`).
- `refused:<Error>`: the policy refused it; `<Error>` is the error's
  short name: one of `Sandboxed`, `AccessOff`, `Remote`, `Locked`,
  `LevelTooLow`, `FeatureOff`, `NotGranted`, `DeniedByUser`, `Excluded`,
  `NeedsConfirmation`, and `AccessDenied` for `AgentAccess1` calls from
  programs other than Settings. The reason is only the error name
  (`Excluded`'s token, e.g. `app:<id>`, is not repeated; the target says
  what was excluded).
- `cancelled`: the user dismissed the prompt or question (`Cancelled`),
  or stopped the agent.

Events (not `Agent1` methods; none of them is a method name):

- `ConsentPrompt`: the consent prompt was answered. `allowed` with
  `grant` (*Allow for this session*, *Always allow*),
  `refused:DeniedByUser` (*Deny*), or `cancelled` (it went away
  unanswered). The call that asked is recorded too, with its own
  outcome.
- `SensitivePrompt`: the sensitive-action prompt was answered;
  `target` is the method, `action` what it does when the prompt said
  more. `allowed` or `refused:DeniedByUser`.
- `Stop`: *Stop* in the indicator; `cancelled`.
- `LogFull`: written once when a day file reaches a cap (below), with
  `agent` and `name` `""`.

`AgentAccess1` (Settings) calls that take access away or clear the log
are recorded, with `agent` the Settings executable, `name` `""`, `method`
the method's name, no `level`, and `allowed` (unknown agent ids too:
the line says what the user asked for):

- `Revoke`, `Forget`: `target` the agent's id as Settings gave it.
- `StopAll`: no `target`.
- `ClearLog`: no `target`; written after the files are removed (so a
  cleared log says it was cleared).

`ListAgents` is not recorded. Calls to `AgentAccess1` that are refused
(`AccessDenied`, a program other than Settings) are recorded, with the
method name.

Examples:

```
{"time":1795082400,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"ListWindows","target":"org.gnome.TextEditor","level":"read-only","outcome":"allowed"}
{"time":1795082410,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"SearchFiles","target":"file:///home/me","level":"read-only","outcome":"allowed","query-length":6}
{"time":1795082420,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"WaitForWindowClosed","target":"window:21 (org.example.Vault)","level":"read-only","outcome":"refused:Excluded"}
{"time":1795082430,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"WaitForWindow","level":"read-only","outcome":"allowed","error":"TimedOut","title-length":5}
{"time":1795082440,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"ConsentPrompt","level":"read-only","outcome":"allowed","grant":"session"}
{"time":1795082450,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"WindowAction","target":"window:21 (org.gnome.TextEditor)","level":"desktop","outcome":"allowed","action":"tile-left"}
{"time":1795082455,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"SensitivePrompt","target":"SetToggle","level":"desktop","outcome":"refused:DeniedByUser","action":"wifi-off"}
{"time":1795082456,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"SetToggle","target":"wifi","level":"desktop","outcome":"refused:DeniedByUser","state":"off"}
{"time":1795082460,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"Open","target":"https://example.org","level":"desktop","outcome":"allowed"}
{"time":1795082470,"agent":"/usr/bin/helix-settings-app","name":"","method":"Revoke","target":"/usr/bin/node /home/me/agent.js","outcome":"allowed"}
{"time":1795082471,"agent":"/usr/bin/helix-settings-app","name":"","method":"StopAll","outcome":"allowed"}
{"time":1795082501,"agent":"/opt/bot","name":"","method":"GetState","level":"read-only","outcome":"refused:AccessOff","count":120,"until":1795082559}
```

### No secrets

A line holds only the keys above. Never recorded: text an agent types
or sends (`Ask`/`Confirm` questions, reminder and notification text,
`SetValue` values, input), clipboard data, notification summaries and
bodies, accessible field values, window titles, answers to questions,
and search queries and title matches. *Helix's choice:* a search query
can say as much as a document ("letter to my divorce lawyer"), so only
its length is recorded. Later methods record only kinds (a MIME type,
a toggle's name) and lengths; the keys and text of synthetic input are
not recorded. An address an agent opens is recorded by its scheme and
host only: its path and query can hold anything (a password reset
token).

### Not filling the disk

*Helix's choice:*

- **Coalescing.** A refusal identical to one recorded less than 60
  seconds before (same `agent`, `method`, `target` and `outcome`) is
  counted instead of written; when the minute is up (checked each
  minute, and when the next identical refusal comes), one line stands
  for the counted calls (`time` the first, `until` the last, `count`).
  An agent hammering a refused method writes about two lines a minute.
  The counts of the last minute are lost if the shell stops. Allowed
  calls are never coalesced.
- **Caps.** Once a day file is 4 MiB, refusals (anything not
  `allowed`) are no longer written that day; at 16 MiB nothing is. A
  `LogFull` line marks each (once per day and shell run).
- Refused calls are recorded with access off too (coalesced): the user
  can see that something tried.

### Retention

Day files are kept for `log-retention-days` days counting today: with
30, today's file and the 29 before it; older day files are removed when
the shell starts, when the local day changes (checked each minute) and
when the setting changes. Only names of the form `YYYY-MM-DD.jsonl`
are pruned. `ClearLog` removes every `*.jsonl` file there.
