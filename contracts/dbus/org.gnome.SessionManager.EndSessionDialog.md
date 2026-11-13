# org.gnome.SessionManager.EndSessionDialog — behavioral notes

Companion to `org.gnome.SessionManager.EndSessionDialog.xml`. Served by the
shell (bus name `org.gnome.Shell`, object
`/org/gnome/SessionManager/EndSessionDialog`); called by the session
manager to confirm ending the session.

## `Open(type, timestamp, seconds_to_stay_open, inhibitor_object_paths)`

Shows (or updates, if already open) the confirmation dialog.

| `type` | Dialog | Confirmation signal |
|--------|--------|---------------------|
| 0 | Log out | `ConfirmedLogout` |
| 1 | Power off | `ConfirmedShutdown` |
| 2 | Restart | `ConfirmedReboot` |

- `timestamp`: event time of the triggering input, for focus (may be 0).
- `seconds_to_stay_open`: countdown. When it runs out with nothing
  blocking, the dialog confirms by itself.
- `inhibitor_object_paths`: `org.gnome.SessionManager.Inhibitor` objects
  that block; the dialog lists their apps and reasons, and confirming then
  means "anyway".
- Helix: when the user already confirmed a restart or power off on the
  lock screen (`org.gnome.desktop.screensaver restart-enabled`), the next
  `Open` of that type emits its confirmation signal at once and shows
  nothing.

## Signals

- `ConfirmedLogout` / `ConfirmedShutdown` / `ConfirmedReboot`: the user
  (or the countdown) confirmed.
- `Canceled`: the user cancelled.
- `Closed`: the dialog went away (after any of the above, or `Close`).

`Close` dismisses the dialog without a decision.
