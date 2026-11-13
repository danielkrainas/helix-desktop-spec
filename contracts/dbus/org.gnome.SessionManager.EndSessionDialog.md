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
| 3 | Restart & Install Updates (Helix) | `ConfirmedReboot` |
| 4 | Restart & Install Upgrade (Helix) | `ConfirmedReboot` |

Types 3 and 4 are Helix's numbering (see "Updates" below); other values
show the Log Out dialog.

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

## Updates (Helix)

The dialog works with PackageKit's offline updates
(`org.freedesktop.PackageKit` on the system bus, object
`/org/freedesktop/PackageKit`, interface
`org.freedesktop.PackageKit.Offline`, as PackageKit documents it):
updates downloaded beforehand and installed during the next boot.
When the dialog opens for types 1–4 it reads the interface's properties
`UpdatePrepared`, `UpdateTriggered`, `UpgradePrepared` and
`PreparedUpgrade` (`name` and `version`).

- Power off (1) and restart (2), with an update prepared (or already
  triggered): the dialog has a check box "Install pending software
  updates", ticked; Space or a click toggles it. Confirming with it
  ticked calls `Trigger("power-off")` (power off) or `Trigger("reboot")`
  (restart) before the confirmation signal; unticked, when an update was
  already triggered, `Cancel()`, so the next boot does not install it.
- 3: "Restart & Install Updates", "The computer will restart and install
  updates in N seconds. Installing can take a while.", button "Restart &
  Install": confirming calls `Trigger("reboot")`. Without a prepared
  update it is the Restart dialog (2).
- 4: "Restart & Install Upgrade", "<name> <version> will be installed
  after the restart, in N seconds. Installing can take a long time.",
  button "Restart & Install": confirming calls `TriggerUpgrade("reboot")`.
  Without a prepared upgrade it is the Restart dialog (2).
- When the call fails, the answer is `Canceled` (nothing ends) and the
  notification "Updates could not be scheduled" ("Nothing was turned off
  or restarted." and the error) is shown.
- The countdown confirms as the user would, with the check box as it is.
- PackageKit is only asked on bare hardware (DRM backend); a nested
  session never reads or schedules the host's updates.

## Signals

- `ConfirmedLogout` / `ConfirmedShutdown` / `ConfirmedReboot`: the user
  (or the countdown) confirmed.
- `Canceled`: the user cancelled.
- `Closed`: the dialog went away (after any of the above, or `Close`).

`Close` dismisses the dialog without a decision.
