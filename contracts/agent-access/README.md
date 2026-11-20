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

Each level includes the ones above it. Notification contents, the
clipboard and the contents of apps' text fields need their own switches
(`notifications`, `clipboard`, `read-text-fields`) on top of the level.

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
| `Ask`, `Confirm` | read-only | | a dialog only the user at the machine answers |
| `AddReminder`, `ListReminders`, `CancelReminder` | read-only | | an agent sees only its own reminders |
| `GetClipboard` | read-only | | needs `clipboard`; not what the shell, Settings, a system prompt's program or an excluded app copied, nor a password manager's secret (see "Clipboard methods") |
| `Open`, `Launch` | desktop | when marked | not excluded apps (also as the app that would open a file) |
| `WindowAction` | desktop | `close`, or when marked | not excluded apps' windows |
| `SwitchWorkspace` | desktop | when marked | |
| `SetToggle` | desktop | `wifi`, `airplane-mode`, or when marked | |
| `SetValue` | desktop | when marked | |
| `Notify` | desktop | when marked | labelled with the agent; at most 5 a minute |
| `SetClipboard` | desktop | when marked | needs `clipboard`; the shell holds the copy for the agent |
| `A11yTree`, `A11yFind` | apps | | not excluded apps' windows, nor while a system prompt is shown |
| `A11yAction` | apps | by the control's name (see "Control apps methods"), `window.close`, or when marked | also not password fields; clipboard actions need `clipboard` |
| `A11ySetValue` | apps | when marked | also not password fields |
| `Screenshot` | screen-and-input | | the compositor draws it; protected windows and shell surfaces black; not of a protected window, nor while a system prompt is shown (see "Screen and input methods") |
| `Input` | screen-and-input | when marked | into one window that has the focus; not protected windows or password fields, nor while a system prompt is shown; the user's own input stops it |

The shell implemented the methods in stages (helixdesktop/desktop#146
to #151); a shell without a method has it absent from introspection and
answers `org.freedesktop.DBus.Error.UnknownMethod`. `GetStatus` and
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
| `notifications`, `clipboard`, `read-text-fields`, `confirm-sensitive` | `b` | the switches of those names |

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
- **Accessible node**: `id` (`s`), `parent` (`s`, the parent's id; `""`
  for the first node of a tree), `role` (`s`, AT-SPI role name as
  at-spi2-core names it: `button`, `text`, `password text`…), `name`,
  `description` (`s`), `states` (`as`, AT-SPI state names: `focused`,
  `checked`, `editable`…), `actions` (`as`, the app's action names),
  `child-count` (`u`), and when they apply: `value` (`s` text, or `d`
  with `min` and `max` (`d`) for a number; never for password fields,
  nor for text fields with `read-text-fields` off), `value-truncated`
  (`b`), `password` (`b`, true: a password field), `value-hidden` (`b`,
  true: a text field whose text was withheld),
  `truncated` (`b`, some of its children were left out), `x`, `y`,
  `width`, `height` (`i`, in its window, with `bounds`). See "Control
  apps methods".
- **Image** (`Screenshot`): `png` (`ay`), `width`, `height` (`i`,
  pixels), `scale` (`d`, pixels per logical pixel), `x`, `y`,
  `logical-width`, `logical-height` (`i`, the rectangle captured, in
  logical pixels: in the layout, or for a window in the window),
  `filled` (`u`, how many windows and surfaces were blacked out in it).
- **Clipboard** (`GetClipboard`): `mime-type` (`s`), and `text` (`s`)
  for text types or `data` (`ay`) otherwise; `types` (`as`), every type
  the clipboard offers (see "Clipboard methods").

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

### Talk to the user methods

The options and arguments of the methods that talk to the user
(helixdesktop/desktop#148), besides the common ones. Malformed values
fail with `InvalidArgs`. Lengths are in characters, after the text is
made plain (below).

| Method | Options and arguments | Returns |
|--------|----------------------|---------|
| `Ask` | `question` (1–500); `choices` (`as`, 2–8 answers of 1–64, no two the same: the user picks one; without it the user types an answer of at most 1,000), `default` (`s`: the answer filled in, at most 1,000; with `choices`, one of them, which has the focus first), `timeout-ms` (`u`, 0 or absent: no limit) | `answer` (`s`: the text typed, or the choice), `choice` (`i`: the choice's index from 0, -1 for a typed answer); `Cancelled`, `TimedOut` |
| `Confirm` | `question` as `Ask`'s; `yes`, `no` (`s`: the buttons' labels, 1–24, not the same; default "Yes" and "No" in the user's language), `timeout-ms` | `yes` (`b`): true for yes, false for no; `Cancelled` (Escape), `TimedOut` |
| `AddReminder` | `time` (Unix seconds, from 60 seconds ago to 366 days ahead; a time just past is shown at once), `text` (1–500) | the reminder's id (`s`) |
| `ListReminders` | | this agent's Reminder records not yet shown, by time |
| `CancelReminder` | `id` | `NotFound` when this agent has no such reminder (another agent's included) |

- **Plain text.** *Helix's choice:* control characters (line breaks
  too) become spaces, characters that reorder text (U+061C, U+200E,
  U+200F, U+202A–U+202E, U+2066–U+2069) are removed, so an agent's text
  cannot rearrange what is shown next to it, and the ends are trimmed.
  Nothing is markup: `<b>` is shown as written. The display name in the
  dialog is cleaned the same way.
- **The dialog.** *Helix's choice:* a shell-modal dialog unlike the
  system's prompts (the consent, sensitive-action, polkit and keyring
  prompts): a band in the indicator's red says "Question from an agent",
  the title names the agent ("“Build bot” asks", its display name, else
  its executable's file name), the next line its program (its id, as
  the consent prompt shows it), the agent's words stand in a quoted
  block, and the last line says Helix did not ask. Below the question:
  a text field with *Cancel* and *Answer*; or a button for each choice,
  then *Cancel*; or the `no` and `yes` buttons. The focus starts in the
  text field, on the `default` choice (else the first), or on `no`.
  Enter in the text field answers (an empty answer too); Escape cancels.
- **Only the user answers.** As the consent prompt ("Consent"): the
  answer travels only inside the shell; the dialog's layer surface has
  the namespace `helix-agent-prompt`, so while it is shown the
  compositor drops remote desktop, EIS and the headless backend's test
  input; accessibility actions on it are refused (setting the text
  field's text too; screen readers still read it, and the keyboard
  works it); keys and clicks in its first second are ignored. The
  price: **a remote desktop user answers at the machine**, as for the
  consent prompt; otherwise an agent could answer its own question
  through remote desktop or EIS.
- **One at a time.** *Helix's choice:* an agent's questions are queued,
  not refused: its next `Ask` or `Confirm` waits until the one before is
  answered (the `timeout-ms` counts while it waits), and questions of
  different agents are shown one after another. A question never covers
  the lock screen or another prompt (consent, sensitive action, polkit,
  keyring, network secrets…): it waits for it; one shown when such a
  prompt comes is put back first in line, and shown again after it
  (with its first second ignored again).
- **How it ends.** The answer; *Cancel* or Escape: `Cancelled`;
  `timeout-ms` passing: the dialog closes, `TimedOut`; locking the
  screen closes it (or the queued question): `Cancelled`, not `Locked`,
  since the user did not answer; the agent losing access meanwhile
  (`enabled` off, a lower `access-level`, *Stop*, a revoke): it closes
  with the error a new call would get; the caller leaving the bus (the
  CLI killed): it closes, `Failed`. While a question is open or queued
  the agent counts as active (the indicator shows it). `interactive`
  false only spares the consent prompt; the question itself is shown,
  since it is what the call asks for.
- **Reminders.** *Helix's choice:* kept in
  `$XDG_STATE_HOME/helix/agent-reminders.json` (mode 0600, written whole
  through a temporary file, never through a symlink, and a rename), so
  they outlive the shell and the session:

  ```json
  {"version": 1, "reminders": [{"id": "3f9a0c2e7b1d4a56", "agent": "/usr/bin/node /home/me/agent.js", "executable": "/usr/bin/node", "name": "Example", "time": 1795082400, "text": "Check the deploy", "added": 1795080600}]}
  ```

  Ids are 16 random hexadecimal digits. An agent may have 20 reminders
  waiting, and add 5 in any minute (more fail with `Failed`). When one
  is added, the shell says so in a notification under "Agent: <agent>"
  (as `Notify`'s): "Reminder set for <time>", the text, and a *Cancel
  Reminder* button, so the user sees what an agent scheduled and can
  undo it. When it is due (looked at when the shell starts, so reminders
  due while it was not running are shown then, at their times, and at
  least every 30 seconds) it is shown as a notification under "Agent:
  <agent>" titled "Reminder" with the text, the "Reminder set"
  notification is taken back, and it is removed. Reminders due while
  `enabled` is off are removed without being shown. Grants do not
  matter when a reminder is shown: the agent had access when it set it.
  Reminders show while the screen is locked as any notification does,
  and Do Not Disturb holds back their banners.
- **Cancelling.** An agent lists and cancels only its own. The user
  cancels one with the *Cancel Reminder* button, or in Settings
  (`AgentAccess1.ListReminders` and `CancelReminder`, "Who may manage
  grants").

### Control apps methods

The options and arguments of the *Control apps* methods
(helixdesktop/desktop#149), besides the common ones. Malformed values
fail with `InvalidArgs`. `window` is a window id as `ListWindows` gives
it; `node` a node id from `A11yTree` or `A11yFind`.

| Method | Options and arguments | Returns |
|--------|----------------------|---------|
| `A11yTree` | `depth` (`u`, levels below the first node, 0: it alone; default and most 64), `max-nodes` (`u`, 1–5,000, default 1,000), `node` (`s`: start there, else at the window), `bounds` (`b`: add `x`, `y`, `width`, `height` in the window) | `nodes` (`aa{sv}`: Accessible node records in tree order, the first where reading started), `window` (`u`), `app-id` (`s`), `truncated` (`b`: nodes were left out) |
| `A11yFind` | `role` (`s`, a role name), `name` (`s`, contained in the name, ignoring case), `states` (`as`, all of them), `limit` (`u`, 1–500, default 50), `timeout-ms` (`u`: wait until one matches; 0 or absent: answer at once) | the matching Accessible node records, in tree order, without children (`[]` when none matches at once); `TimedOut` |
| `A11yAction` | `node`; `action`: one of the node's `actions`, or `default` (its first action), `focus` (the keyboard focus), `select` (select it in the list, tabs or menu holding it) | `NotSupported` when the node has no such action or the app cannot do it |
| `A11ySetValue` | `node`; `value` (`v`): a string replaces an editable text's text (or, for a node with a number and no editable text, is that number), a number (`d`, `i`, `u`, `x`) sets the node's number, within its `min` and `max` | `NotSupported` when the node has neither; `InvalidArgs` outside the range |

- **The shell is the only client.** *Helix's choice:* the shell
  connects to the accessibility bus itself (`AT_SPI_BUS_ADDRESS`, else
  `org.a11y.Bus.GetAddress` on the session bus) and reads and acts on
  the app's objects there; agents never get an object's bus name or a way
  to reach it.
- **Which tree is a window's.** *Helix's choice:* by the window's
  process, never by its title alone. The compositor tells the shell
  each window's process (its Wayland client's credentials; for an X11
  window the process it names) and, when the app said with
  `gtk_surface1.set_a11y_properties` (GTK 4), its bus name and the
  window's object path. The told object is used when the process owning
  that name on the accessibility bus is the window's; a told name of
  another process is refused. Otherwise the registry's applications are
  asked for the one whose bus connection is the window's process: none,
  or more than one, is `NotSupported`; of its top-level objects that are
  shown, the one, or the one named as the window is titled (needed when
  the process has several windows). Anything else is ambiguous and
  refused (`NotSupported`), never guessed. Apps whose accessibility
  connection is not their own process (a sandbox's D-Bus proxy) cannot
  be driven.
- **Node ids.** A node's id is its object path on the app's connection,
  shortened to its last part when it shares the window's object's
  directory (GTK 4: `2bfe25b2_b780_4e52_9c1e_6d0a3f8b7e10`), else the
  whole path. It names the same control for as long as the app keeps
  that object (GTK, Qt: the widget's lifetime), across tree reads, not
  across app restarts; a gone node is `NotFound`. Every action first
  checks that the node is at exactly that path (its parent gives it at
  its place: some toolkits answer for paths they never gave) and walks
  from it up its parents to the window's object: a node of another
  window, another app, or none is `NotFound`. Only nodes of the
  app's own bus connection are read (objects embedded from other
  processes are left out, and their holder marked `truncated`).
- **What is read.** Roles, names, descriptions, states and action names
  as the app gives them; a number's value, minimum and maximum; and the
  text of editable texts (and of other texts whose text is not their
  name), at most 4,096 characters (`value-truncated`); text fields' text
  only with `read-text-fields` (below). *Helix's choice:* other texts'
  contents are given: driving apps means reading what they show, the
  user chose *Control apps* knowing it, and apps whose contents must not
  be read belong in `excluded-apps` (a password manager, banking,
  messaging). Unlike notifications, which every app sends, the agent
  reads one window it names.
- **Text fields.** *Helix's choice:* what the user types is input, not
  UI, and is withheld unless the user turns `read-text-fields` on
  (default off). A text field is a node with the `EditableText`
  interface or the state `editable`, and without a number (sliders and
  spin buttons keep their `value`, `min` and `max`). With the switch
  off, its record has `value-hidden` true and no `value` or
  `value-truncated`, not even for an empty field; a name or description
  that is exactly the field's text (some toolkits name a field after what
  is in it) is given as `""`, and `A11yFind`'s `name` matches only names
  as given, so a field's text cannot be found by searching for it.
  Labels' and other non-editable texts' names and values are given
  either way. `A11ySetValue` still sets a field's text (writing is not
  reading), and nothing returns it afterwards. The switch is read on
  every call, and again by an `A11yFind` waiting with `timeout-ms` each
  time it looks.
- **Password fields.** A node is one when its role is `password text`,
  or it is an editable text whose text is all mask characters (`•`, `●`,
  `∗`, `*`…; GTK 4 reports an entry with hidden characters as plain
  `text`) or whose name contains "password", "passphrase", "passcode",
  "PIN" or "secret" (or the words for them in the user's language).
  AT-SPI has no password state; toolkits that know the field is one use
  the role. A password field's record has `password` true and no
  `value`, its text is never read (not even masked), its children are not
  read, and `A11yAction` (`focus` too), `A11ySetValue` and `A11yTree`'s
  `node` on it or anything inside it fail with `Excluded`
  (`password-field`). An empty field with hidden characters that no
  name marks cannot be told from a plain one.
- **Protected surfaces.** Before anything is read, in order: the
  window's app id is checked as any target (Settings: `settings`;
  `excluded-apps`: `app:<id>`); while a system prompt is shown (the
  shell's polkit, keyring, network secret, Bluetooth pairing, consent,
  sensitive-action and question dialogs, and its other permission
  prompts) everything fails with `system-prompt`; then the window's
  process: the shell's own (its panels, prompts and dialogs are never a
  target; `shell`), Settings' program `helix-settings-app` (`settings`),
  the shell's, compositor's, greeter's, session's and portals' programs
  (`shell`), the system's prompt programs (`gcr-prompter`, polkit
  agents, `pinentry*`, `ssh-askpass*`, KWallet: `system-prompt`), and a
  process that shows a window of an excluded app or Settings, runs an
  excluded app's program, or runs in an excluded app's systemd unit
  (`app:<id>`, `settings`). The lock screen and the greeter have no
  windows agents can name, and nothing runs while the screen is locked;
  each action checks the lock and the system prompts again just before
  it is done (after the sensitive-action prompt).
- **Clipboard.** Actions that use the clipboard (`clipboard.*`, and
  actions named `copy`, `cut` or `paste`) need the `clipboard` switch
  (`FeatureOff`): pasting into a field and reading it would read the
  clipboard.
- **Sensitive actions.** *Helix's choice:* `A11yAction` is sensitive
  when the agent marks it, for `window.close`, `selection.delete` and
  `text.clear`, and when the control's name or description has, as a
  whole word, one of: delete, remove, erase, discard, trash, send,
  submit, post, publish, pay, purchase, buy, order, checkout, transfer,
  donate, uninstall, format, wipe, reset, empty, revoke, quit, sign out,
  log out, shut down, or their translations in the user's language.
  `focus` is never sensitive. `A11ySetValue` is sensitive when marked.
  The shell cannot know what a control does; agents should mark what
  they know to be sensitive. The prompt names the control by its name
  and role and the app ("Control: “Delete All” (button) in
  org.gnome.Nautilus"), shown to the user only.
- **Limits.** A tree read stops after `max-nodes` nodes or 20 seconds
  (`truncated`); each call to the app waits at most 3 seconds (`Failed`
  when it does not answer). `A11yFind` reads up to 5,000 nodes; with
  `timeout-ms` it looks again every quarter second, counts as running
  (the indicator), and ends with the error a new call would get when the
  agent loses access, a system prompt appears or the window closes.

### Clipboard methods

The options and arguments of the clipboard methods
(helixdesktop/desktop#151), besides the common ones. Both need the
`clipboard` switch (`FeatureOff`, step 7) on top of their level:
reading is *Read only*, setting *Control the desktop*. Malformed values
fail with `InvalidArgs`.

| Method | Options and arguments | Returns |
|--------|----------------------|---------|
| `GetClipboard` | `type` (`s`): `text` or a plain text type (`text/plain`, `text/plain;charset=utf-8`, `UTF8_STRING`…) for plain text, `major/*` (`image/*`) for the first type of that kind offered, else exactly that type (ignoring case); absent: plain text, else `image/png`, else the first type offered. `types-only` (`b`): only what it offers | the Clipboard record; with `types-only`, only `types`. `NotFound` when the clipboard is empty or does not offer the type |
| `SetClipboard` | `mime_type`: `text/plain` (`""` and `text/plain;charset=utf-8` too), `text/html`, `text/uri-list` or `image/png`; `data`: the content, 1 byte to 16 MiB, UTF-8 for text types, a PNG image for `image/png` | nothing. `NotSupported` for other types, `InvalidArgs` for bad data |

- **Through the shell.** Agents never take part in the clipboard
  themselves. The shell asks the compositor what the clipboard offers
  and which client set it, and has the compositor hand it the content;
  `SetClipboard` gives the compositor the data, which then owns the
  clipboard *for the agent* (Wayland and X11 apps paste it until
  something else is copied). Plain text is offered as
  `text/plain;charset=utf-8`, `text/plain`, `UTF8_STRING`, `STRING`
  and `TEXT`; other types as themselves.
- **What is read is what was checked.** *Helix's choice:* the
  compositor numbers each clipboard change and serves only the change
  the shell checked; when the clipboard changes in between, the call
  fails with `Failed` ("the clipboard changed while it was read"), never
  with the newer content.
- **Where the content came from.** *Helix's choice:* the compositor
  records who set the clipboard: the Wayland client that set it (the
  app id of one of its windows, the focused one first, and its process),
  for an X11 app the focused X11 window when it took the clipboard, the
  shell, an agent (`SetClipboard`), or a remote desktop session.
  `GetClipboard` (also with `types-only`) fails with `Excluded` when the
  content came from the shell (`shell`: its text fields, copied search
  results and screenshots: the shell's prompts never reach an agent,
  and a screenshot only through `Screenshot`), from
  Settings (`settings`), from an excluded app (`app:<app id>`), or from a
  process that is the shell's, Settings', a system prompt's program
  (pinentry, gcr-prompter, polkit agents…) or an excluded app's (by its
  other windows, its program or its systemd unit, as for the *Control
  apps* methods). An agent's copy, a remote desktop client's, and an
  app whose window and process are not known are read: the `clipboard`
  switch is the user's explicit choice, and refusing what cannot be
  traced would refuse most X11 copies.
- **Password managers.** Content offered with the type
  `x-kde-passwordManagerHint` whose value is `secret` (as KeePassXC and
  KDE's apps mark passwords) is refused with `Excluded`
  (`password-field`); a hint that cannot be read within a second counts
  as `secret`. Apps' own password fields do not copy (GTK, Qt); the
  compositor cannot see which field an app copied from.
- **Primary selection.** *Helix's choice:* not supported. The primary
  selection is whatever text the user last selected, without copying
  it; reading it would read what the user never chose to share, and
  setting it would change what a middle click pastes.
- **Lock and prompts.** Both methods fail with `Locked` while the screen
  is locked and with `Excluded` (`system-prompt`) while a system prompt
  is shown, checked when the call comes and again just before the
  content is read or set (after the sensitive-action prompt). `SetClipboard`
  is sensitive only when marked; with `confirm-sensitive` on, the user
  confirms before the clipboard changes.
- **Limits.** *Helix's choice:* at most 16 MiB either way (`Failed` when
  the clipboard holds more; `InvalidArgs` for more data); the app that
  copied has 5 seconds to send it (`TimedOut`). Text types are given as
  `text` when they are UTF-8 (`STRING` and `TEXT` may be Latin-1 and are
  read as such), else as `data`.
- **Nothing recorded.** The activity log records the type read or set as
  a common kind (`mime-type`: `text/plain`, `text/html`,
  `text/uri-list`, `image/png`…, else `other`: apps name their own
  types) and the length in bytes (`data-length`), and for `GetClipboard`
  the app the content came from (`target`); never the content.

### Screen and input methods

The options and arguments of the *Screen and input* methods
(helixdesktop/desktop#150), besides the common ones. Malformed values fail
with `InvalidArgs`. `window` is a window id as `ListWindows` gives it;
positions are logical pixels from the window's top-left corner (the
`x`, `y` of its Window record, its title bar not included).

| Method | Options and arguments | Returns |
|--------|----------------------|---------|
| `Screenshot` | `window` (`u`: that window alone, whether or not it is shown), or `monitor` (`s`: a connector as `ListMonitors` gives it), else every monitor; `x`, `y`, `width`, `height` (`i`, all four: only that area, in the window, the monitor, or the layout) | the Image record. `NotFound` for an unknown window or monitor, `InvalidArgs` for an area that is empty, not inside, or more than 16,384 pixels either way |
| `Input` | `events` (`aa{sv}`, 1–500, carried out in order); `window` (`u`: into that window, which is focused first; default: the focused window) | nothing; the errors below |

Events (`type` and its keys):

| `type` | Keys | Does |
|--------|------|------|
| `move` | `x`, `y` (`d` or `i`) | moves the pointer there |
| `click` | `button` (`s`: `left` (default), `middle`, `right`, `back`, `forward`), `x`, `y` (moves there first), `count` (`u`, 1–3) | presses and releases the button `count` times |
| `down`, `up` | `button` (as `click`, with `x`, `y` for `down`), or `key` (`s`, a key name) | presses or releases it; whatever an `Input` call holds is let go when it ends |
| `scroll` | `dx`, `dy` (`i`, wheel steps, at most 100 either way; positive: right, down), `x`, `y` | scrolls where the pointer is (moving there first) |
| `key` | `keys` (`s`): a key combination, `+`-separated modifiers (`ctrl`, `alt`, `shift`, `super`, `altgr`) and a key | presses the modifiers, presses and releases the key, releases the modifiers |
| `text` | `text` (`s`): any characters but control characters (line feed is Return, tab is Tab); at most 10,000 in all | types it |
| `wait` | `ms` (`u`, at most 5,000) | waits |

Key names are XKB keysym names (`Return`, `Escape`, `F4`, `Left`,
`eacute`; letters as themselves, `a` and `A` being different keys; any case
for the rest), and `enter`, `esc`, `del`, `ins`, `pgup`, `pgdn`,
`backspace`, `space` and the modifiers' names. An unknown type is
`NotSupported`.

- **The compositor draws, the agent never sees more.** *Helix's
  choice:* the shell asks the compositor for the picture
  (`helix_screenshot_v1` version 8, `contracts/wayland/README.md`) and
  tells it what may be shown: the windows it cleared, and the shell
  surfaces (layer namespaces) agents may see. Everything else is drawn as
  opaque black (filled, never blurred) in its place in the stacking order,
  with its popups and title bar, so what is above it still shows and a
  window that opened after the shell decided is black. The pointer is
  never drawn.
- **What is black.** Excluded apps' windows (`app:<id>`), Settings',
  the shell's own, the system prompt programs' (`gcr-prompter`, polkit
  agents, `pinentry*`, `ssh-askpass*`, KWallet), and every window of a
  process that shows one of those, runs an excluded app's program or runs
  in its systemd unit (as for the *Control apps* methods; windows whose
  process is not known are judged by their app id). Windows the compositor
  has no window list id for (X11 override-redirect menus and tooltips).
  Of the shell, everything but the wallpaper (`helix-background`), the
  desktop icons, the top bar and its tooltips, the dock, the on-screen
  displays (volume, workspace switch, monitor labels, click type): its
  prompts and dialogs (polkit, keyring, network, Bluetooth, consent,
  sensitive actions, questions, the run dialog), notifications and the
  notification list, the overview and the window switcher (they show
  windows' pictures), the launcher, menus, the on-screen keyboard (it
  shows what is typed), the screenshot UI. The lock screen and the
  greeter are never captured: nothing runs while the screen is locked,
  and the compositor refuses captures while it is.
- **Password fields.** A picture cannot tell where a password field is;
  apps mask what is typed into them (dots), and a screenshot shows the
  dots. *Helix's choice:* nothing more is done for pictures.
- **Window screenshots.** A protected window's picture is refused
  outright (`Excluded`), never sent black. A window's picture is its
  surfaces alone (cropped to its window geometry), drawn whether or not
  it is shown (on another workspace, behind others).
- **Refused** while the screen is locked (`Locked`) and while a system
  prompt is shown (`Excluded`, `system-prompt`), checked when the call
  comes and again just before the compositor draws.
- **Size.** *Helix's choice:* at most 16,384 pixels either way, drawn at
  the highest scale of the monitors captured; a PNG of more than 64 MiB
  fails (`Failed`: take a monitor, a window or an area).
- **Input goes to one window.** *Helix's choice:* an `Input` call names
  one window (or takes the focused one); it is checked as a target
  (excluded apps, Settings, the shell's, a system prompt program's, and
  windows of a process that shows one of those: `Excluded`), focused
  first when it was named and is not focused, and checked again. The
  shell surfaces (top bar, dock, overview, notifications, prompts) are
  never a target: input goes to an app's window or nowhere.
- **Checked before every event, by the compositor.** The compositor
  carries the events out (`helix_agent_input_v1`) and, just before each
  key, button, motion and scroll, checks that the screen is not locked,
  no Agent Access prompt is shown, the window is still mapped and has the
  keyboard focus (also through its menus and popups), for pointer events
  that the point is inside the window and the window's own surface is
  there (not another window, a title bar or a shell surface over it), and
  for keys and text that the app's focused text field is not a password
  or PIN field (below). The first that fails stops the call: nothing
  after it is sent. The shell checks the grant, the level, the lock and
  system prompts before each event too, and positions against the
  window's size (`InvalidArgs`). Focus moving to another window, a shell
  dialog or a system prompt in the middle stops it.
- **No compositor keys.** *Helix's choice:* synthetic events go straight
  to the window's app, never through the compositor's key bindings, the
  shell's keys, click-to-focus, window moving or title bar buttons: an
  agent's Super, Alt+Tab, Ctrl+Alt+F*n*, Super+L or Alt+F4 reach the
  app as keys (which may close its window: Alt+F4 is the app's then) and
  never switch windows, workspaces or virtual terminals, lock the screen
  or open the overview.
- **Keys and text.** `key`, `down` and `up` keys are keys of the user's
  keyboard layout (the key that gives that keysym at its first or second
  level, with Shift for the second), so apps see real key codes; a key
  the layout does not have is `NotSupported`. `text` is typed with a
  keymap made for the text (as a virtual keyboard does), a key for each
  character, with no modifiers: any Unicode character, whatever the
  layout, Caps Lock or keys the user or the call holds. Apps that read key
  codes rather than the characters (some games, terminals' raw modes,
  some X11 apps) may see made-up keys for typed text. About 250
  characters a second.
- **Password fields.** *Helix's choice:* the compositor knows a text
  field is a password or PIN field when the app says so over
  `zwp_text_input_v3` (its content purpose; GTK, Qt, Firefox and
  Chromium do for their password fields). A `key`, `down`/`up` key or
  `text` event while one has the focus stops the call with `Excluded`
  (`password-field`). Apps that do not use `zwp_text_input_v3` (X11 apps)
  cannot be checked.
- **The user's input stops it (the hard stop).** *Helix's choice:*
  while an `Input` call runs, anything the user does with an input device
  (a key or button press, pointer motion, scrolling, a touch, a tablet
  pen; from the machine's own devices, a remote desktop session or EIS)
  stops it at once, even in the middle of a text: `Cancelled`
  (`user-input`). For 2 seconds after, new `Input` calls are refused with
  `Cancelled` (`cooldown`). Releases do not count (the key that started
  the agent may still be going up). What the call held is let go.
- ***Stop* stops it.** The indicator's *Stop*, Settings' `Revoke`,
  `Forget` and `StopAll`, and turning Agent Access off stop running
  `Input` calls at once: `Cancelled` (`stopped`).
- **Sensitive.** *Helix's choice:* `Input` is sensitive only when
  marked: the shell cannot know what a click or Enter does (sends a
  message, buys something), and asking about every keystroke would teach
  the user to say yes. Agents should mark input that sends, deletes or
  pays. `Screenshot` is never sensitive.
- **The indicator.** While an agent takes a screenshot or sends input,
  and for 3 seconds after, the indicator says an agent is controlling the
  screen (another icon, a white ring; its menu and accessible name say
  so).
- **Errors.** Besides the policy's: `NotFound` for no focused window or
  a window that went away, `Failed` when the window lost the focus or the
  point is outside it or covered, `NotSupported` for a key the layout
  does not have, `Locked`, `Excluded` (`system-prompt`,
  `password-field`), `Cancelled` (`user-input`, `cooldown`, `stopped`).
- **Nothing recorded.** `Screenshot`'s line has `scope` (`screen`,
  `monitor`, `window`), `target` (`screen`, `monitor:<connector>` or
  `window:<id> (<app id>)`), `width`, `height` and `filled-count`;
  never the picture. `Input`'s has the window as `target`, `event-count`,
  counts of kinds (`move-count`, `click-count`, `button-count`,
  `scroll-count`, `key-count`, `shortcut-count`), `text-length`, and
  `done-count` (steps carried out before it ended); never the text, key
  names or positions. A key combination with Ctrl, Alt, Super or AltGr
  counts as a shortcut, without saying which.

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
| `Cancelled` | the user dismissed `Ask` or `Confirm`, or the screen locked while it was shown or queued; `Input` stopped by the user | `Input`: `user-input` (the user's own input), `cooldown` (moments after), `stopped` (*Stop*, Settings, access off) |
| `TimedOut` | a wait or question ran out of time | |
| `NotFound` | no such window, app, node or reminder | |
| `NotSupported` | a name or action this shell, this computer or the app does not support (e.g. a toggle without hardware, an app that does not let its focus be moved) | why, in English (e.g. `the app does not support this`); free text, not a token |
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
locked. While an agent takes a screenshot or sends input, and for 3
seconds after, it says an agent is controlling the screen (see "Screen
and input methods"). Its menu lists the active agents (display name and executable,
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
- **Password fields.** Accessible nodes with the role `password text`,
  and editable texts the shell takes for one (see "Control apps
  methods"), have no `value`; acting on them, setting them, or sending
  input while one has focus fails with `Excluded` (`password-field`).
  Screenshots show what the app shows of them (masked text); typing into
  one stops `Input` (see "Screen and input methods").
- **System prompts.** While the shell shows a polkit, keyring, network
  secret, Bluetooth pairing, Agent Access consent or sensitive-action
  prompt (or an agent's question, or another permission prompt of the
  shell's), `A11y*`, `Screenshot`, `Input` and the clipboard methods fail
  with `Excluded` (`system-prompt`), as do apps' windows of the system's
  prompt programs and what they copied; the shell's own UI is never a
  target (`shell`), nor is what the shell copied.
- **Settings.** *Helix's choice:* every window of the Settings app
  (`org.helixdesktop.Settings`) is excluded (`settings`), because the
  shell cannot tell which page a window shows and the Agent Access page
  must never be reachable; Settings also holds other security pages.
- **User exclusions:** the apps in `excluded-apps`
  (`app:<app id>`).

Excluded apps' windows are left out of `ListWindows`, `ListApps` and
`WaitForWindow`, their notifications out of `ListNotifications`, and
they are black in screenshots (and refused as a window screenshot or
`Input`'s window); launching or opening them
(also opening a file whose default app is one), any method naming
one of their windows, and reading what they copied, fails with
`Excluded`. App ids are compared
without a `.desktop` suffix.

## Settings take effect at once

The shell reads the settings on every call and watches them: turning
`enabled` off ends every session grant and denial; lowering
`access-level` narrows every grant; turning a switch off refuses the
next call that needs it. Revoking an agent (Settings, or *Stop*)
ends its grant at once. A call already waiting (a wait, a question, a
prompt) when its agent loses access ends with the error the agent would
now get; a question the lock closed ends `Cancelled` ("Talk to the user
methods").

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
- `ListReminders()`: every agent's reminders not yet shown (Reminder
  records with `agent`, `executable` and `name`), by time.
- `CancelReminder(id)`: cancels any agent's reminder.
- `RemindersChanged` is emitted when a reminder is added, shown or
  cancelled.

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
| `target` | string | What the call read or acted on, absent for none: an app id as the agent gave it (`org.gnome.Maps`), a window as `window:<id>` and, once known, ` (<app id>)` (`window:21 (org.gnome.TextEditor)`), a file or folder as a `file://` URI, an address as its scheme and host only (`https://example.org`, `mailto:`), a workspace as `workspace:<index>`, a quick setting or value by its name (`wifi`, `volume`), a reminder as `reminder:<id>` (`AddReminder` once added, `CancelReminder`, the `Reminder` event); for `GetClipboard`, the app id of the app the content came from, when known; for `Screenshot`, `screen`, `monitor:<connector>` or the window; for Settings' `Revoke` and `Forget`, the agent's id. Never a window title. Control characters are removed and it is cut at 256 characters. |
| `level` | string | The access level the call needed (`read-only`, …); for a consent prompt the level asked, for *Stop* the level the agent held. Absent for calls needing none. |
| `outcome` | string | `allowed`, `refused:<Error>` or `cancelled` (below). |
| `error` | string | For an `allowed` call that then failed: the error's short name (`TimedOut`, `NotFound`, `NotSupported`, `Failed`, `InvalidArgs`). Absent when it succeeded. |
| `<kind>-length` | integer | The length in characters of something the agent sent, instead of it: `query-length` (`SearchFiles`), `title-length` (a window match's `title`, a notification's title), `body-length` (a notification's body), `question-length` (`Ask`, `Confirm`), `text-length` (`AddReminder`, the text `A11ySetValue` sets, the text
`Input` types), `name-length` (`A11yFind`'s `name`); later stages add others. |
| `choice-count` | integer | `Ask` with `choices`: how many. |
| `scope` | string | `Screenshot`: `screen`, `monitor` or `window`. |
| `width`, `height` | integer | `Screenshot`: the picture's size in pixels. |
| `filled-count` | integer | `Screenshot`: how many windows and surfaces were blacked out in it. |
| `event-count`, `done-count` | integer | `Input`: how many events it had; how many steps (moves, presses, releases, texts, waits) were carried out. |
| `move-count`, `click-count`, `button-count`, `scroll-count`, `key-count`, `shortcut-count` | integer | `Input`: how many of each kind (absent for none); clicks counted one by one, `button-count` for `down`/`up` buttons, `key-count` for keys without Ctrl, Alt, Super or AltGr, `shortcut-count` for combinations with them. |
| `uri-count` | integer | `Launch`: how many files or addresses it handed the app (absent for none). |
| `action` | string | `WindowAction`: the action (`tile-left`, `close`); `A11yAction`: the action when it is a common one (`click`, `activate`, `toggle`, `default`, `focus`, `select`, `window.close`, `clipboard.copy`…), else `other` (apps name their actions); `A11ySetValue`: `set-text` or `set-value`; a `SensitivePrompt`: what the prompt said the call does, where it said more than the method (`wifi-off`, `close`). |
| `role` | string | `A11yAction`, `A11ySetValue`: the role of the node acted on (`button`, `password text`), also when refused; `A11yFind`: the role looked for. |
| `node-count` | integer | `A11yTree`: how many nodes it returned; `A11yFind`: how many matched. |
| `hidden-count` | integer | `A11yTree`, `A11yFind`: how many of the nodes returned had their text withheld (`read-text-fields` off); absent for none. Never which. |
| `state` | string | `SetToggle`: `on` or `off`. |
| `mime-type` | string | `GetClipboard`, `SetClipboard`: the type read or set as a common kind (`text/plain` for every plain text type, `text/html`, `text/uri-list`, `image/png`, `image/jpeg`, `image/gif`, `image/svg+xml`, `application/pdf`, `x-kde-passwordManagerHint`), else `other`. |
| `data-length` | integer | `GetClipboard`: how many bytes it returned; `SetClipboard`: how many the agent sent. |
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
- `Reminder`: a reminder came due and was shown (`allowed`), was due
  while `enabled` was off and removed unshown (`refused:AccessOff`), or
  the user cancelled it with the notification's button (`cancelled`);
  `target` is `reminder:<id>`, `agent` and `name` the agent that set it,
  no `level`.
- `LogFull`: written once when a day file reaches a cap (below), with
  `agent` and `name` `""`.

`AgentAccess1` (Settings) calls that take access away or clear the log
are recorded, with `agent` the Settings executable, `name` `""`, `method`
the method's name, no `level`, and `allowed` (unknown agent ids too:
the line says what the user asked for):

- `Revoke`, `Forget`: `target` the agent's id as Settings gave it.
- `CancelReminder`: `target` `reminder:<id>` as Settings gave it.
- `StopAll`: no `target`.
- `ClearLog`: no `target`; written after the files are removed (so a
  cleared log says it was cleared).

`ListAgents` and `ListReminders` are not recorded. Calls to `AgentAccess1` that are refused
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
{"time":1795082462,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"Ask","level":"read-only","outcome":"allowed","question-length":29,"choice-count":2}
{"time":1795082464,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"Confirm","level":"read-only","outcome":"cancelled","question-length":12}
{"time":1795082466,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"AddReminder","target":"reminder:3f9a0c2e7b1d4a56","level":"read-only","outcome":"allowed","text-length":16}
{"time":1795084200,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"Reminder","target":"reminder:3f9a0c2e7b1d4a56","outcome":"allowed"}
{"time":1795082468,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"A11yTree","target":"window:21 (org.gnome.TextEditor)","level":"apps","outcome":"allowed","node-count":143}
{"time":1795082469,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"A11ySetValue","target":"window:21 (org.gnome.TextEditor)","level":"apps","outcome":"allowed","text-length":17,"role":"text","action":"set-text"}
{"time":1795082469,"agent":"/usr/bin/node /home/me/agent.js","name":"Example","method":"A11ySetValue","target":"window:21 (org.gnome.TextEditor)","level":"apps","outcome":"refused:Excluded","text-length":5,"role":"password text"}
{"time":1795082470,"agent":"/usr/bin/helix-settings-app","name":"","method":"Revoke","target":"/usr/bin/node /home/me/agent.js","outcome":"allowed"}
{"time":1795082471,"agent":"/usr/bin/helix-settings-app","name":"","method":"StopAll","outcome":"allowed"}
{"time":1795082501,"agent":"/opt/bot","name":"","method":"GetState","level":"read-only","outcome":"refused:AccessOff","count":120,"until":1795082559}
```

### No secrets

A line holds only the keys above. Never recorded: text an agent types
or sends (`Ask`/`Confirm` questions and choices, reminder and notification text,
`SetValue` values, input), clipboard data, notification summaries and
bodies, accessible field values, window titles, answers to questions,
and search queries and title matches; for the *Control apps* methods,
node ids, names and values (a window is recorded as `window:<id> (<app
id>)`, a node by its role, a set text by its length, a set number not at
all). *Helix's choice:* a search query
can say as much as a document ("letter to my divorce lawyer"), so only
its length is recorded. Later methods record only kinds (a MIME type,
a toggle's name) and lengths; the keys, text and positions of synthetic
input are not recorded, nor screenshots' pixels. An address an agent opens is recorded by its scheme and
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
