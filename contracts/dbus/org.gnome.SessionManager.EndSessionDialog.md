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

*GNOME does not document this; Helix's choice:* values beyond 0–2 (GNOME
has update and upgrade dialogs too, opened by callers it does not name).
Types 3 and 4 are Helix's numbering (see "Updates" below); other values
show the Log Out dialog. Nothing in Helix opens 3 or 4 yet; the session
manager may, for software apps.

*Helix's choice:* only the session manager (the owner of
`org.gnome.SessionManager`) may call `Open` and `Close`; others get
`org.freedesktop.DBus.Error.AccessDenied`. The session manager takes the
answer signals only from the dialog's owner (the unique name owning
`org.gnome.Shell` when it called `Open`), so no other client can confirm
or cancel for the user.

- `timestamp`: event time of the triggering input, for focus (may be 0).
- `seconds_to_stay_open`: countdown. When it runs out with nothing
  blocking, the dialog confirms by itself.
- `inhibitor_object_paths`: `org.gnome.SessionManager.Inhibitor` objects
  that block; the dialog lists their apps and reasons, each with its app's
  icon (24 logical px; the icon of the desktop file named by the
  inhibitor's app id, else a generic one) before its name, and confirming
  then means "anyway".
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
  updates", ticked; a click toggles it. It is a keyboard stop of its
  own, before Cancel (Tab and Shift+Tab, with the focus ring when it
  has the focus), and Space toggles it there; Enter there answers with
  the dialog's default button. Confirming with it
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
- Tested against a fake PackageKit on a private system bus only; no real
  PackageKit has scheduled updates through it (helixdesktop/desktop#48).

## Signals

- `ConfirmedLogout` / `ConfirmedShutdown` / `ConfirmedReboot`: the user
  (or the countdown) confirmed.
- `Canceled`: the user cancelled.
- `Closed`: the dialog went away (after any of the above, or `Close`).

`Close` dismisses the dialog without a decision.
