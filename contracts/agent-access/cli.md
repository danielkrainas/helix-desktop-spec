# `helix-agent`

The command-line front end to Agent Access (`README.md`). It is a thin
client of `org.helixdesktop.Agent1` with no policy of its own: every
command is one or a few method calls, and every refusal comes from the
shell. Installed as `/usr/bin/helix-agent` (helix-desktop), with a man
page `helix-agent(1)` and completions for bash, zsh and fish.

## Usage

```
helix-agent [OPTIONS] <COMMAND> [ARGS]
```

Global options (before or after the command):

| Option | Meaning |
|--------|---------|
| `--json` | JSON output (the default when standard output is not a terminal) |
| `--text` | readable text (the default on a terminal) |
| `--no-ask` | never make the shell ask the user; where it would have to, fail (exit 12 or 18) |
| `--sensitive` | mark this action sensitive (the user confirms it when `confirm-sensitive` is on) |
| `--timeout <SECONDS>` | how long to wait for the shell's answer (default 30; `0`: no limit). Waits and questions use their own `--timeout` |
| `-h`, `--help` | help for the command |
| `-V`, `--version` | the CLI's version and the shell's Agent Access version |

The display name the user sees comes from `HELIX_AGENT_NAME` in the
environment (`README.md`, "Who is calling"); the CLI has no option for
it, since identity never comes from the agent.

## Commands

"Level" is what the shell requires (`README.md`, "Access levels").

| Command | Method | Level |
|---------|--------|-------|
| `status` | `GetStatus` | none |
| `request [--level <LEVEL>]` | `RequestAccess` | the one asked for |
| `state` | `GetState` | read-only |
| `windows [--app <APP_ID>] [--focused]` | `ListWindows` | read-only |
| `apps [--running]` | `ListApps` | read-only |
| `workspaces` | `ListWorkspaces` | read-only |
| `monitors` | `ListMonitors` | read-only |
| `notifications [--limit <N>]` | `ListNotifications` | read-only |
| `files recent [--limit <N>] [--type <MIME>]` | `RecentFiles` | read-only |
| `files search <QUERY> [--since <WHEN>] [--until <WHEN>] [--type <MIME>] [--in <FOLDER>] [--limit <N>]` | `SearchFiles` | read-only |
| `wait window [--app <APP_ID>] [--title <TEXT>] [--focused] [--timeout <SECONDS>]` | `WaitForWindow` | read-only |
| `wait closed <WINDOW> [--timeout <SECONDS>]` | `WaitForWindowClosed` | read-only |
| `ask <QUESTION> [--choice <TEXT>]... [--default <TEXT>] [--timeout <SECONDS>]` | `Ask` | read-only |
| `confirm <QUESTION> [--yes <LABEL>] [--no <LABEL>] [--timeout <SECONDS>]` | `Confirm` | read-only |
| `remind add <WHEN> <TEXT>` | `AddReminder` | read-only |
| `remind list` | `ListReminders` | read-only |
| `remind cancel <ID>` | `CancelReminder` | read-only |
| `clipboard get [--type <MIME>] [--output <FILE>]` | `GetClipboard` | read-only, `clipboard` switch |
| `clipboard types` | `GetClipboard` (`types-only`) | read-only, `clipboard` switch |
| `clipboard set [<TEXT>] [--type <MIME>]` (standard input without `TEXT`) | `SetClipboard` | desktop, `clipboard` switch |
| `open <URI-OR-PATH> [--app <APP_ID>]` | `Open` | desktop |
| `launch <APP_ID> [<URI-OR-PATH>...]` | `Launch` | desktop |
| `window <ACTION> <WINDOW> [ARGS]` | `WindowAction` | desktop |
| `workspace switch <INDEX>` | `SwitchWorkspace` | desktop |
| `toggle <NAME> [on\|off]` | `SetToggle` | desktop |
| `set <NAME> <VALUE>` | `SetValue` | desktop |
| `notify <TITLE> [<BODY>] [--icon <ICON>] [--urgency low\|normal]` | `Notify` | desktop |
| `a11y tree <WINDOW> [--depth <N>] [--node <NODE>] [--max-nodes <N>] [--bounds]` | `A11yTree` | apps |
| `a11y find <WINDOW> [--role <ROLE>] [--name <TEXT>] [--state <STATE>]... [--limit <N>]` | `A11yFind` | apps |
| `a11y wait <WINDOW> [--role <ROLE>] [--name <TEXT>] [--state <STATE>]... [--timeout <SECONDS>]` | `A11yFind` | apps |
| `a11y do <WINDOW> <NODE> <ACTION>` | `A11yAction` | apps |
| `a11y press <WINDOW> <NODE>` / `a11y focus <WINDOW> <NODE>` / `a11y select <WINDOW> <NODE>` | `A11yAction` | apps |
| `a11y set <WINDOW> <NODE> <VALUE> [--number]` | `A11ySetValue` | apps |
| `screenshot [--window <WINDOW> \| --monitor <CONNECTOR>] [--area <X,Y,WIDTH,HEIGHT>] [--output <FILE>]` | `Screenshot` | screen-and-input |
| `input key <KEYS>...` / `input type <TEXT>` / `input click <X> <Y> [--button <BUTTON>] [--count <N>]` / `input move <X> <Y>` / `input scroll <DX> <DY> [--at <X,Y>]` / `input drag <X1> <Y1> <X2> <Y2> [--button <BUTTON>]` / `input sequence <STEP>...`, each with `[--window <WINDOW>]` | `Input` | screen-and-input |
| `log [--agent <ID>] [--since <WHEN>]` | none (reads the log file) | none |
| `help [<COMMAND> \| exit-codes]` | | |
| `completions <bash\|zsh\|fish>` | | |

- `LEVEL`: `read-only`, `desktop`, `apps`, `screen-and-input`.
- `WINDOW`: a window id from `windows` (a number).
- `WHEN`: an ISO 8601 date or date and time (local time unless it has
  an offset), or a relative time: `+10m`, `+2h`, `-1d`, `yesterday`,
  `today`.
- `window` actions: `focus`, `close`, `minimize`, `maximize`,
  `unmaximize`, `restore`, `fullscreen`, `unfullscreen`, `tile-left`,
  `tile-right`, `move <X> <Y>`, `resize <WIDTH> <HEIGHT>`,
  `to-workspace <INDEX>` (sent as `WindowAction`'s options `x`, `y`,
  `width`, `height`, `workspace`). `close` is sensitive.
- `toggle` names: `dark-style`, `do-not-disturb`, `night-light`, `wifi`,
  `airplane-mode`, `mute`; without `on`/`off` the setting flips (read
  with `GetState` first; `not-supported` when the state does not have
  it). `wifi` and `airplane-mode` are sensitive.
- `set` names: `volume`, `brightness`; `VALUE` is a percentage
  (`0`–`100`, `volume` up to `150` when over-amplification is on), sent
  as `SetValue`'s 0–1 value.
- `open` and `launch` take paths (relative ones are made absolute
  against the working directory, `~` is the home folder), `file://`
  URIs, and `http`, `https` and `mailto` addresses.
- `a11y`: `NODE` is a node id from `a11y tree` or `a11y find`
  (`README.md`, "Control apps methods"). `tree` sends `--depth`,
  `--node`, `--max-nodes` and `--bounds` as the options of those names;
  `find` sends `--role`, `--name`, `--state` (repeatable, as `states`)
  and `--limit`. `wait` is `find` with `limit` 1 and its `--timeout`
  (default 60 seconds; `0`: as long as it takes) as `timeout-ms`; the CLI
  waits for the shell's answer that long plus the global `--timeout`.
  `press` sends the action `default` (the node's first), `focus` and
  `select` those actions. `set` sends `VALUE` as a string, or with
  `--number` as a number (`d`). Text output of `tree` is an indented
  outline (role, name, value, states, actions, `#id`; “(password
  field)” for `password`, “(hidden)” (translated) for `value-hidden` in
  place of the value); of `find` and `wait` a line per node. The text
  output of `status` has a line for `read-text-fields` (“Reading text
  fields: on/off”) when the shell reports it.
- `clipboard get` sends `--type` as `type` (`README.md`, "Clipboard
  methods": `text/plain`, `text/html`, `image/*`…; default plain text,
  else a picture). Its text output is the content as it is, with no line
  break added, so it can be piped; binary content (no `text`) is written
  to standard output only when that is not a terminal (otherwise exit 2
  with “The clipboard holds binary data; use --output FILE or redirect
  standard output”). `--output` writes the content (the text's UTF-8,
  or the data) to the file; relative paths are against the working
  directory. `clipboard types` prints one type per line.
- `clipboard set` sends `TEXT` (or, without it, all of standard input,
  at most 16 MiB: more is exit 2) as `data`, and `--type` (`text/plain`,
  `text/plain;charset=utf-8`, `text/html`, `text/uri-list` or
  `image/png`; others are exit 2) as `mime_type` (`""` without it).
- `input`: positions are logical pixels in the window, from its
  top-left corner; `--window` (after the subcommand) sends the input into
  that window, focusing it first (default: the focused window); each
  command is one `Input` call (`README.md`, "Screen and input methods").
  `key` takes key combinations such as `ctrl+c`, `alt+F4`, `Return`
  (XKB key names; modifiers `ctrl`, `alt`, `shift`, `super`, `altgr`),
  each a `key` event, pressed in turn. `type` sends one `text` event.
  `click` sends `click` with `x`, `y`, `--button` (`left`, `middle`,
  `right`, `back`, `forward`) as `button` and `--count` (1–3) as `count`.
  `move` sends `move`; `scroll` sends `scroll` (`DX`, `DY` in wheel
  steps; `--at X,Y` as its `x`, `y`). `drag` sends `down` (the button, at
  `X1 Y1`), `move` (`X2 Y2`) and `up`. `sequence` sends one event per
  step, in order: `move X Y`, `click [BUTTON] [X Y]`, `double-click
  [BUTTON] [X Y]` (`count` 2), `mouse-down [BUTTON] [X Y]`, `mouse-up
  [BUTTON]` (`down`/`up` with `button`, default `left`), `key KEYS`,
  `key-down KEY`, `key-up KEY` (`down`/`up` with `key`), `scroll DX DY`,
  `type TEXT`, `wait MS`; what it holds is let go when it ends. Output is
  `{}` (nothing in text).
- `screenshot` sends `--window`, `--monitor` and `--area` (`X,Y,WIDTH,HEIGHT`,
  in the window, the monitor or the screen) as the options `window`,
  `monitor` and `x`, `y`, `width`, `height`. `--output` writes the PNG
  there (text output: “Saved a W×H screenshot to FILE”, and “Blacked out:
  N” when something was); without it, JSON output has the PNG in base64
  and text output refuses (exit 2: “A screenshot is binary data; use
  --output FILE or redirect standard output”) unless standard output is
  not a terminal, where it writes the PNG.
- `--limit`: a whole number from 1 (the shell gives at most 200 files
  and 50 notifications). `--type` may be `major/*` (`image/*`).
- `files search --in` takes a folder path (relative ones are made
  absolute against the working directory, `~` is the home folder) or a
  `file://` URI; `--since` and `--until` are `WHEN`s, sent as Unix
  seconds.
- `wait`'s `--timeout` is how long to wait (default 60 seconds; `0`: as
  long as it takes); the CLI waits for the shell's answer that long plus
  the global `--timeout` (no limit when either is `0`). `wait window`
  returns at once when a matching window is open; `--title` matches
  anywhere in the title, ignoring case.
- `ask` and `confirm` send `QUESTION` as `question`, `--choice` (2–8,
  in order) as `choices`, `--default`, `--yes` and `--no` as the options
  of those names (`README.md`, "Talk to the user methods"). Their
  `--timeout` is how long the user has to answer (default 300 seconds;
  `0`: as long as it takes), sent as `timeout-ms`; the CLI waits for the
  shell's answer that long plus the global `--timeout` (no limit when
  either is `0`). Text output of `ask` is the answer; of `confirm`,
  nothing (the exit code says it).
- `remind add` sends `WHEN` as Unix seconds (`+10m`: ten minutes from
  now) and `TEXT`; its text output says the id and the local time.
  `remind list` prints one line per reminder: id, local time and text.
- `log` reads `$XDG_STATE_HOME/helix/agent-activity/` itself; it needs
  no access because the log is the user's own file. `--agent` keeps
  the entries whose `agent` is the id given; `--since` those whose
  `time` (Unix seconds, or ISO 8601 text) is at or after it.
- `completions` prints a script made from the same command table as the
  parser; packages install them (bash:
  `/usr/share/bash-completion/completions/helix-agent`, zsh: `_helix-agent`
  in the distribution's vendor directory, fish:
  `/usr/share/fish/vendor_completions.d/helix-agent.fish`).

*Helix's choice:* the CLI and the shell ship together, so the CLI knows
which methods its shell has. A command whose method the shell does not
have (none since helixdesktop/desktop#150) still parses its arguments and
has its `--help` (which says so), and then fails with `not-supported`
(exit 23) without calling the shell; a newer CLI talking to an older
shell gets the same from `UnknownMethod`.

## `--help`

Every level has `--help` (and `help <command>`), in this shape:

```
$ helix-agent --help
Let an agent read and act on your Helix session.

Usage: helix-agent [OPTIONS] <COMMAND>

Status:
  status         Show whether agent access is on and what this agent may do
  request        Ask the user for access now
Read (Read only):
  state          Focused window, workspace and quick settings
  windows        Open windows
  apps           Installed and running apps
  workspaces     Workspaces
  monitors       Monitors
  notifications  Recent notifications
  files          Recent files and file search
  wait           Wait for a window to appear or close
Talk to the user (Read only):
  ask            Ask the user a question
  confirm        Ask the user yes or no
  remind         Reminders for the user
Act (Control the desktop):
  open           Open a file or URL in its app
  launch         Start an app
  window         Focus, move, tile or close a window
  workspace      Switch workspaces
  toggle         Turn a quick setting on or off
  set            Set volume or brightness
  notify         Show the user a notification
  clipboard      Read or set the clipboard (needs Clipboard access)
Apps (Control apps):
  a11y           Read and use an app's controls
Screen and input (Screen and input):
  screenshot     Take a screenshot
  input          Send keys, text, clicks and scrolling
Other:
  log            Show what agents did
  help           Help for a command, or "help exit-codes"
  completions    Shell completions

Options:
  --json             JSON output (default when not on a terminal)
  --text             Readable text (default on a terminal)
  --no-ask           Never ask the user; fail instead
  --sensitive        Mark this action sensitive
  --timeout <SECONDS>  How long to wait for the shell [default: 30]
  -h, --help         Print help
  -V, --version      Print version

Agent access is controlled in Settings › Privacy & Security › Agent Access.
```

A command's help has: one line saying what it does, `Usage:`, a
`Needs:` line with the level (and switch), its arguments and options,
and for commands with JSON output a pointer to the record keys
(`README.md`, "Records"). The help texts are translated.

## Output

**Text** is for people: tables and sentences, translated, not for
parsing.

**JSON** is one JSON document on standard output, ending in a newline,
always an object:

| Command | Success output |
|---------|----------------|
| `status`, `request` | the status dictionary (`README.md`, "Methods") |
| `state` | the State record |
| `windows` | `{"windows": [Window...]}` |
| `apps` | `{"apps": [App...]}` |
| `workspaces` | `{"workspaces": [Workspace...]}` |
| `monitors` | `{"monitors": [Monitor...]}` |
| `notifications` | `{"notifications": [Notification...]}` |
| `files recent`, `files search` | `{"files": [File...]}` |
| `wait window` | the Window record |
| `wait closed` | `{}` |
| `ask` | `{"answer": "<text>", "choice": <index or -1>}` |
| `confirm` | `{"yes": true\|false}` |
| `remind add` | `{"id": "<id>"}` |
| `remind list` | `{"reminders": [Reminder...]}` |
| `clipboard get` | the Clipboard record (`data` in base64; with `--output`, `"path"` instead of `"text"` or `"data"`) |
| `clipboard types` | `{"types": [...]}` |
| `notify` | `{"id": <id>}` |
| `toggle` | `{"name": "<NAME>", "on": true\|false}` (the state it set) |
| `a11y tree` | `{"window": <id>, "app-id": "<id>", "truncated": true\|false, "root": <node>}`: the shell's flat list as a tree, each Accessible node with `children` (a list of nodes) and without `parent` |
| `a11y find` | `{"nodes": [Accessible node...]}` |
| `a11y wait` | the Accessible node found |
| `screenshot` | the Image record (`png` in base64; with `--output`, `"path"` instead of `"png"`) |
| `log` | `{"entries": [...]}` (the log's own lines) |
| other commands | `{}` |

Records are the shell's dictionaries with the same keys
(`README.md`, "Records"): D-Bus integers and doubles become JSON
numbers, strings and string lists stay, byte arrays become base64
strings. Keys the CLI does not know are passed through.

On failure, JSON output is (still on standard output, nothing on
standard error):

```json
{"error": {"code": "access-off", "exit-code": 10, "message": "Agent access is turned off (Settings › Privacy & Security › Agent Access)", "dbus-error": "org.helixdesktop.Agent1.Error.AccessOff"}}
```

`message` is the text output's message (translated), `code` and
`exit-code` are from the table below, and `dbus-error` is the D-Bus
error name when there was one. Text output prints `helix-agent: ` and
the message on standard error.

## Exit codes and messages

| Exit | `code` | From | Message |
|------|--------|------|---------|
| 0 | | | success (`confirm`: the user said yes) |
| 1 | `failed` | `Failed`, anything else | the shell's message |
| 2 | `usage` | bad arguments | the argument parser's message |
| 3 | `no-shell` | the bus or `org.gnome.Shell` unreachable, or no `Agent1` | Cannot reach the Helix shell (is a Helix session running?) |
| 10 | `access-off` | `AccessOff` | Agent access is turned off (Settings › Privacy & Security › Agent Access) |
| 11 | `level-too-low` | `LevelTooLow` | This needs the “{level}” access level (Settings › Privacy & Security › Agent Access) |
| 12 | `not-granted` | `NotGranted` | The user has not allowed this agent, and could not be asked |
| 13 | `denied` | `DeniedByUser` | The user denied this agent access |
| 14 | `excluded` | `Excluded` | see below |
| 15 | `locked` | `Locked` | The screen is locked; agents can't act until it is unlocked |
| 16 | `remote` | `Remote` | Agent access from remote sessions is turned off (Settings › Privacy & Security › Agent Access) |
| 17 | `sandboxed` | `Sandboxed` | Agent access is not available to sandboxed apps |
| 18 | `needs-confirmation` | `NeedsConfirmation` | This action needs the user's confirmation; run it without --no-ask to ask |
| 19 | `feature-off` | `FeatureOff` | Clipboard access is turned off (Settings › Privacy & Security › Agent Access) |
| 20 | `no` | `Cancelled`, or `confirm` answered no | The user said no / The user cancelled the question / Stopped: the user took over the keyboard or pointer (`user-input`) / The user is using the keyboard or pointer; try again in a moment (`cooldown`) / The user stopped the agent (`stopped`) |
| 21 | `timed-out` | `TimedOut` | Timed out |
| 22 | `not-found` | `NotFound` | No such window / app / item: {what} |
| 23 | `not-supported` | `NotSupported` | Not supported here: {reason} (“Not supported here” when the shell gave none) |
| 23 | `not-supported` | `UnknownMethod`, a command the CLI knows the shell lacks | Not supported by this Helix session |

`{reason}` is the shell's `NotSupported` message as it is (English):
it says whether the session, the computer or the app cannot do it
(`the app does not let its focus be moved over accessibility` for
`a11y focus` on a GTK 4 app). `help exit-codes` describes 23 as “Not
supported by this Helix session, this computer or the app”.
`{level}` is the level's name as Settings shows it: “Read only”,
“Control the desktop”, “Control apps”, “Screen and input”. `Excluded`
messages, by the error's message token: `app:<id>`: “{app name} is
excluded from agent access”; `settings`: “Agents can't use Settings”;
`password-field`: “Agents can't use password fields”; `system-prompt`:
“Agents can't act while a system prompt is open”; `shell`: “Agents
can't act on the shell itself”. `confirm` answered no exits 20 with
`{"yes": false}` as its output and no error object; a dismissed
question is the error `no` with the message “The user cancelled the
question”. Messages are translated in text output; JSON `code` values
never are.
