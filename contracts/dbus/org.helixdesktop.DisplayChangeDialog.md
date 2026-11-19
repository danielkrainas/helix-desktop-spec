# org.helixdesktop.DisplayChangeDialog — behavioral notes

Companion to `org.helixdesktop.DisplayChangeDialog.xml`. Helix-private; served by
the shell (bus name `org.gnome.Shell`, object
`/org/helixdesktop/DisplayChangeDialog`) and called by the compositor after a
persistent `ApplyMonitorsConfig` (see `org.gnome.Mutter.DisplayConfig.md`).

- `Open(seconds)`: ask the user whether to keep the new display settings,
  counting down `seconds`. Calling it again while open restarts the
  question.
- `Kept`: the user kept the change.
- `Reverted`: the user asked to go back, or the countdown reached zero.
- `Close()`: dismiss the question without an answer (no signal follows).

Exactly one of `Kept` or `Reverted` follows each `Open` that is not closed.

Only the compositor (the owner of `org.gnome.Mutter.DisplayConfig`) may
call `Open` and `Close`; others get `org.freedesktop.DBus.Error.AccessDenied`.
The compositor takes `Kept` and `Reverted` only from the shell it asked
(the unique name owning `org.gnome.Shell` at `Open`).
