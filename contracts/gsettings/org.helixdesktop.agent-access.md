# `org.helixdesktop.agent-access`

Helix's Agent Access settings (not a GNOME schema; GNOME has no
counterpart). Helix installs it as
`org.helixdesktop.agent-access.gschema.xml` (in
`$datadir/glib-2.0/schemas/`), and this table is its contract. The
Settings page Privacy & Security › Agent Access writes these keys; the
shell reads them on every agent call and follows changes at once.
Behavior: `contracts/agent-access/README.md`.

Path: `/org/helixdesktop/agent-access/`

| Key | Type | Default |
|-----|------|---------|
| `enabled` | `b` | `false` |
| `access-level` | `s` | `'read-only'` |
| `remember-mode` | `s` | `'ask'` |
| `notifications` | `b` | `false` |
| `clipboard` | `b` | `false` |
| `read-text-fields` | `b` | `false` |
| `confirm-sensitive` | `b` | `true` |
| `allow-remote` | `b` | `false` |
| `excluded-apps` | `as` | `[]` |
| `log-retention-days` | `u` | `30` |

- `enabled`: the master switch. Off, the shell refuses every agent call
  except `GetStatus` (error `AccessOff`). Turning it off also ends every
  session grant, stops agents' input at once and forgets this
  session's denials; remembered
  (*Always allow*) grants are kept for when it is turned on again.
- `access-level`: the most an agent may be granted, one of
  `read-only`, `desktop`, `apps`, `screen-and-input` (the schema's
  choices), each including the ones before it. Settings offers the
  first three as one choice and `screen-and-input` separately, behind a
  warning. A grant never exceeds the level the user approved in the
  consent prompt, and never exceeds this key: lowering it narrows every
  grant at once, raising it widens none (an agent is asked again when it
  needs more). An unknown value reads as `read-only`.
- `remember-mode`: `ask` (default) asks on an agent's first use in each
  session, and the consent prompt offers *Allow for this session*;
  `remember` also offers *Always allow*, and remembered agents are not
  asked. Under `ask`, remembered grants are ignored but kept (Settings
  still lists them). An unknown value reads as `ask`.
- `notifications`: agents may read notification titles and bodies. Off,
  `ListNotifications` gives only each notification's app and time.
- `clipboard`: agents may read (*Read only*) and set (*Control the
  desktop*) the clipboard, and use apps' copy, cut and paste actions
  (*Control apps*). Off, the clipboard methods and those actions fail
  with `FeatureOff`. Even on, what the shell, Settings, system prompts
  and excluded apps copied, and password managers' secrets, are not
  read (`contracts/agent-access/README.md`, "Clipboard methods").
- `read-text-fields`: agents controlling apps (`apps`) get what is in
  editable text fields. Off, `A11yTree` and `A11yFind` give such fields
  without their text (`value-hidden` instead of `value`), and a name or
  description that is the field's text is left out too; labels and other
  texts that are not fields, and the numbers of sliders and spin
  buttons, are given either way, and `A11ySetValue` still sets a field's
  text (`contracts/agent-access/README.md`, "Control apps methods").
  Summary "Let agents read text fields".
- `confirm-sensitive`: the shell asks the user before every sensitive
  action (see `contracts/agent-access/README.md`, "Sensitive actions").
- `allow-remote`: agents running in a remote login session (SSH) may
  use Agent Access. Off, they are refused with `Remote`.
- `excluded-apps`: app ids (desktop file names without `.desktop`)
  agents may never read or control, in addition to the hard exclusions.
  Their windows are left out of every list and refused as targets.
- `log-retention-days`: days the activity log keeps entries; range
  1–365 (the schema's range). Day files are kept for that many days
  counting today, and pruned at once when it changes
  (`contracts/agent-access/README.md`, "Activity log").

Keys missing from an older installed schema read as the defaults above
(`org.helixdesktop.shell.md`, "Older installed schemas"); a missing
schema reads as all defaults, so Agent Access is off.
