# org.gnome.SessionManager — behavioral notes

Companion to `org.gnome.SessionManager.xml`. Records value semantics that the
introspection XML cannot express.

## `Logout(u mode)`

| Value | Name            | Meaning                                         |
|-------|-----------------|-------------------------------------------------|
| 0     | normal          | Log out, showing a confirmation if needed       |
| 1     | no confirmation | Log out without asking the user to confirm      |
| 2     | force           | Log out regardless of confirmation              |

These are the three values in use by existing GNOME clients. No other values
are defined; implementations should reject any other value with a D-Bus error
rather than guessing.

## Inhibit flags (`Inhibit`, `IsInhibited`, `InhibitedActions`)

`flags` is a bitmask. These are the bits in use by existing GNOME clients and
by `org.freedesktop.portal.Inhibit` (which uses the same values):

| Bit | Value | Inhibits                  |
|-----|-------|---------------------------|
| 0   | 1     | logout (and shutdown / reboot) |
| 1   | 2     | user switching            |
| 2   | 4     | suspend                   |
| 3   | 8     | the session being marked idle |
| 4   | 16    | auto-mounting of media    |

Unknown bits are carried through unchanged (they still count for
`IsInhibited` and appear in `InhibitedActions`), but carry no meaning.

`IsInhibited(flags)` is true when any active inhibitor shares a bit with
`flags`. `InhibitedActions` is the bitwise OR of all active inhibitors' flags.

## Ending the session and inhibitors

- `Logout(0)` and `Logout(1)` are blocked while any inhibitor has the logout
  bit (1). With a confirmation UI the user may choose to override; without one
  the call fails.
- `Logout(2)` (force) ignores inhibitors.
- `Shutdown` and `Reboot` end the session like `Logout(0)` and are blocked by
  the logout bit in the same way.
- Inhibitors without the logout bit (e.g. idle-only) never block ending the
  session.

## `Setenv(s variable, s value)`

Only valid during the initialization phase (before `Initialized`). The
variable becomes part of the session's application launch environment:

- it is added to the D-Bus activation environment
  (`org.freedesktop.DBus.UpdateActivationEnvironment`) before `Setenv`
  returns, so services the bus starts from then on see it;
- it is set in the systemd user manager's environment too
  (`org.freedesktop.systemd1.Manager.SetEnvironment`, when that service
  is on the session bus), so user units started from then on see it.
  The same goes for the environment of Helix's session configuration.
  When the session ends, Helix unsets there the variables it set
  (`UnsetEnvironment`), so a later session does not inherit them;
- session components started after the call inherit it.

Processes that are already running keep their environment. Setting a
variable again replaces its value.

## `GetLocale(i category)`

`category` is the platform C library's `LC_*` constant (e.g. `LC_MESSAGES`).
The reply is the session manager's current locale for that category, as the C
library reports it. An unknown category is an error.

## `InitializationError(s message, b fatal)`

Only valid during the initialization phase (before `Initialized`). Reports a
failure starting a session component. If `fatal` is true the session ends.

## `RestoreSupported` and session restore

When false, clients must not use `RegisterRestore`, `DeletedInstanceIds` or
`UnregisterRestore`. These are a private interface between the session
manager and xdg-desktop-portal; apps use the portal.

- `RegisterRestore(app_id, dbus_name)` registers an app instance for save
  and restore: apps registered when the session ends are started again at
  the next login. It returns a `reason`, a stable `instance_id` and
  `cleanup_ids`:
  - `reason` 0: restore no state; 1: a normal launch (restore what is
    reasonable, e.g. preferences but not history); 2: a restore after the
    app or the whole session crashed (restore carefully); 3: a restore
    after the session ended cleanly (restore as much as possible).
  - `instance_id` includes a part identifying this session manager and
    session, so it never collides with other session managers' ids.
  - `cleanup_ids` are ids that will never be used again; the app deletes
    their data and then reports them with `DeletedInstanceIds`.
- `DeletedInstanceIds(app_id, ids)`: the app deleted those instances'
  data; the session manager forgets them.
- `UnregisterRestore(app_id, instance_id)`: the app exited cleanly and
  needs no restore.

### Helix

- `RestoreSupported` is true. Registrations are kept in
  `$XDG_STATE_HOME/helix/restore.toml` (`~/.local/state/helix/...`).
- An instance is restorable when it was still registered at the end of
  the session (reason 3), or when its app left the bus without
  unregistering, or the whole session ended without the session manager
  ending it (reason 2). Each restorable instance is handed out once, to
  the next `RegisterRestore` of its app; otherwise the app gets a new id
  and reason 1.
- At login, once the session's components are up, every app with
  restorable instances is started once per instance, through its desktop
  file (`<app_id>.desktop`).
- Unregistered instances become cleanup ids, returned to the app's next
  `RegisterRestore` until it reports them deleted.
- `org.gnome.desktop.session` `save-restore` (true without the key):
  when false, no apps are started at login, instances still registered
  when the session ends are given up instead of saved, and a
  `RegisterRestore` gives up its app's instances left from earlier
  sessions (they become cleanup ids; the app gets a new id and reason 1).
  The value is read at each of these moments. Whether GNOME also skips
  restoring after a crash when it is off is not documented; Helix does.
- Ids look like `helix-<session>-<n>`, `<session>` random per login.

## Autostart

Apps start with the session through autostart files: desktop files
(`Type=Application`) in the `autostart` directory under
`$XDG_CONFIG_HOME` and each of `$XDG_CONFIG_DIRS` (freedesktop.org Desktop
Application Autostart Specification). A file in an earlier directory hides
files of the same name in later ones.

GNOME adds keys to these files (not in the freedesktop.org
specifications), among them:

- `AutostartCondition`: when to run the app;
  `GSettings <schema> <key>` runs it while that boolean key is true.
- `X-GNOME-Autostart-enabled=false`: do not start it.

### Helix

- Autostart runs only when the session config's `[session]` section says
  `autostart = true` (Helix's default config does).
- Once the session's components are up (before apps are restored), each
  autostart file is started unless it has `Hidden=true` or
  `X-GNOME-Autostart-enabled=false`, its `OnlyShowIn` names none of the
  desktops in `XDG_CURRENT_DESKTOP` (or its `NotShowIn` names one), its
  `TryExec` program is not found, or its condition does not hold.
- Conditions: `GSettings <schema> <key>` (a boolean key of an installed
  schema; followed: the app is started when the key becomes true and
  sent SIGTERM when it becomes false), `if-exists <file>` and
  `unless-exists <file>` (a path under `$XDG_CONFIG_HOME`, checked once).
  A file with any other condition is not started.
- `Exec` is run directly with the components' environment; its field
  codes are dropped.
- `X-GNOME-Autostart-Phase`: entries in a phase before `Applications`
  (`EarlyInitialization`, `PreDisplayServer`, `DisplayServer`,
  `Initialization`, `WindowManager`, `Panel`, `Desktop`) start first;
  the others (`Applications`, no phase, or a phase Helix does not know)
  1 second later (at once when no entry is early). Helix does not wait
  for early apps to register.
- `X-GNOME-Autostart-Delay`: the entry starts that many more seconds
  later (at most 3600).
- `X-GNOME-AutoRestart` `true`: when the app exits with a failure (a
  non-zero status or a signal) it is started again 1 second later, at
  most 5 times within a minute; an app that ends cleanly, or that a
  condition stopped, is not restarted.

## Client and inhibitor objects

`RegisterClient` returns `/org/gnome/SessionManager/Client<n>`; that object
implements `org.gnome.SessionManager.ClientPrivate` (see below). Each
inhibitor is exported at `/org/gnome/SessionManager/Inhibitor<cookie>`
implementing `org.gnome.SessionManager.Inhibitor`: `GetAppId`, `GetReason`,
`GetFlags`, and `GetClientId` (the inhibiting app's client object, or `/`
if it has none). `GetInhibitors` lists these paths.

## Ending the session

`Logout`, `Shutdown` and `Reboot` start the end-of-session sequence and
return; `SessionOver` is emitted when it completes.

1. **Confirmation.** In normal mode (`Logout(0)`, `Shutdown`, `Reboot`) the
   session manager asks the shell's
   `org.gnome.SessionManager.EndSessionDialog` (see its notes) to confirm,
   listing the inhibitors that block (logout flag). With mode 1 (no
   confirmation) the dialog is only shown if something blocks. Mode 2
   (force) skips confirmation and ignores inhibitors and client answers.
   If no dialog is available, a blocked request fails with an error instead.
   Cancelling stops here and the session keeps running.
   A forced logout (`Logout(2)`) requested while a confirmation is pending
   takes over: the dialog is closed and the session ends as forced.
2. **Query.** Every registered client gets `QueryEndSession(flags)` (flag
   1 = forced) and must answer `EndSessionResponse(is_ok, reason)` within
   one second; silence counts as OK. A client answering `false` blocks like
   an inhibitor (with its reason): the dialog is shown again listing it,
   unless forced. If the user cancels, clients that were queried get
   `CancelEndSession`.
3. **End.** Every client gets `EndSession(flags)` and has ten seconds to
   clean up and answer `EndSessionResponse`.
4. **Stop.** Clients get `Stop`, then `SessionOver` is emitted and the
   session manager exits.
5. **Power.** After `Shutdown`, once the session's processes are stopped,
   the session manager asks logind to power off (`PowerOff(false)`, see
   `org.freedesktop.login1.md`); after `Reboot`, to restart
   (`Reboot(false)`). If logind is unavailable or refuses, the session has
   still ended.

### When the bus goes away (Helix)

If the session manager's connection to the session bus closes (the bus
daemon exits), no client can reach it any more: the session ends at once,
without the steps above and without `SessionOver` (there is no bus to
emit it on). The session's components are then stopped as at any other
end: SIGTERM, and SIGKILL after five seconds. When the session manager
gets SIGTERM, it emits `SessionOver` and leaves the bus for at most two
seconds before stopping the components, so a bus that stops answering
cannot keep them running. Components also get SIGTERM if the session
manager itself dies (Linux's parent-death signal), so they never outlive
it. How gnome-session behaves in these cases is not documented here.

## `Suspend`

Asks logind to suspend (`Suspend(false)`) and returns; the session keeps
running. Fails with an error if logind is unavailable or refuses (for
example because something blocks sleep).

While any inhibitor has the suspend bit (4), the session manager holds a
logind `sleep` inhibitor in `block` mode, so the system does not sleep
until the last such inhibitor goes away.

## `CanShutdown`, `CanReboot`, `CanSuspend`

Return the availability of the action, for showing it in a UI (not for
automatic power management): 0 unavailable, 1 normally available but
completely blocked by an inhibitor right now, 2 available after
authentication, 3 available without authentication.

### Helix

They ask logind (`CanPowerOff`, `CanReboot`, `CanSuspend`): `yes` is 3,
`challenge` 2, anything else (and no logind) 0. `CanSuspend` is 1 instead
of 2 or 3 while an inhibitor has the suspend bit, as suspending is then
blocked. Logout inhibitors do not change `CanShutdown` / `CanReboot`, as
the end-session dialog lets the user go ahead anyway.

## `SessionIsActive`

Follows the `Active` property of the logind session the session manager
runs in (`GetSession($XDG_SESSION_ID)`, or `GetSessionByPID` of its own
process when the variable is unset), with `PropertiesChanged` on every
change. Without logind it is true.
