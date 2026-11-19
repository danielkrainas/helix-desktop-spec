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
| `Open`, `Launch` | desktop | when marked | |
| `WindowAction` | desktop | when marked | |
| `SwitchWorkspace` | desktop | when marked | |
| `SetToggle` | desktop | `wifi`, `airplane-mode`, or when marked | |
| `SetValue` | desktop | when marked | |
| `Notify` | desktop | when marked | |
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
  (`s`), `workspace` (`u`, index from 0), `monitor` (`s`, connector),
  `focused` (`b`), `x`, `y`, `width`, `height` (`i`, logical pixels),
  `state` (`s`: `normal`, `maximized`, `fullscreen`, `minimized`,
  `tiled-left`, `tiled-right`), `document` (`s`, URI of the open
  document, only when the app tells it).
- **App**: `id` (`s`), `name` (`s`), `running` (`b`), `windows` (`u`).
- **Workspace**: `index` (`u`), `active` (`b`), `windows` (`u`).
- **Monitor**: `connector` (`s`), `name` (`s`), `x`, `y`, `width`,
  `height` (`i`, logical), `scale` (`d`), `primary` (`b`).
- **State** (`GetState`): `focused-window` (`u`, 0 for none),
  `active-workspace` (`u`), `dark-style`, `do-not-disturb`,
  `night-light`, `wifi`, `airplane-mode`, `muted` (`b`), `volume` (`d`,
  0–1, above 1 when over-amplified), `brightness` (`d`, 0–1; absent
  without a controllable screen).
- **Notification**: `id` (`u`), `app-id` (`s`), `app-name` (`s`),
  `time` (`x`, Unix seconds), and with `notifications` on, `summary` and
  `body` (`s`); without it, `redacted` (`b`, true).
- **File**: `uri` (`s`), `name` (`s`), `mime-type` (`s`), `modified`
  (`x`), `size` (`t`).
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
and the level) with *Stop* for each, and opens Settings at Privacy &
Security, where the Agent Access page is.

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
they are blacked out in screenshots; launching or opening them, and any
method naming one of their windows, fails with `Excluded`. App ids are
compared without a `.desktop` suffix.

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
- `ClearLog()`: empties the activity log (helixdesktop/desktop#144).
- `AgentsChanged` is emitted when the list changes.

Unknown agent ids are not an error (nothing to take away).

## Activity log

The shell records every `Agent1` call except `GetStatus`
(helixdesktop/desktop#144): time, agent id and display name, method,
target (app id, window title, file URI; no content), level, and outcome
(`allowed`, `refused:<error>`, `cancelled`). It never records secrets:
no typed text, clipboard data, notification contents, field values or
question answers, only their kind and length. The log is kept in
`$XDG_STATE_HOME/helix/agent-activity/` (directory 0700), one file per
day (`YYYY-MM-DD.jsonl`, 0600, one JSON object per line), and files older
than `log-retention-days` are removed. Settings reads it directly and
clears it with `ClearLog`; `helix-agent log` reads it for the user.
