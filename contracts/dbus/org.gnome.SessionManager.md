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

### The graphical session target

Services that belong to a graphical session (portal backends, for
example) are tied to systemd's `graphical-session.target`, which cannot
be started by hand: a desktop's own target brings it up by binding it.
Helix installs `helix-session.target` (`BindsTo=graphical-session.target`,
after `graphical-session-pre.target`). Once the session's components are
up and before autostart apps start, Helix starts the target named by the
session configuration's `systemd_target` (`helix-session.target` in the
installed configuration); when the session ends it stops it, before
unsetting the environment. Without a systemd user manager, or with no
`systemd_target` (nested test sessions), nothing is started.

### The login's identity

The D-Bus bus and the systemd user manager often outlive a login, so they
can still hold the previous desktop's values (an X11 session's
`XDG_SESSION_TYPE=x11`, its `DESKTOP_SESSION`, `XDG_MENU_PREFIX`). Apps
the bus starts read these: some pick their display backend from
`XDG_SESSION_TYPE`, and menus are looked up under `XDG_MENU_PREFIX`. So at
startup, before any component starts, Helix publishes as if through
`Setenv` the variables naming this login, with the values its own
process environment has: `XDG_SESSION_TYPE`, `XDG_SESSION_ID`,
`XDG_SESSION_CLASS`, `XDG_SEAT`, `XDG_VTNR`, `DESKTOP_SESSION`,
`GDMSESSION`, `XDG_SESSION_DESKTOP` and `XDG_CURRENT_DESKTOP` (those the
session configuration's `[environment]` names take its values). Ones the
process environment lacks are left alone. `XDG_MENU_PREFIX` becomes
`gnome-` (the application menu GNOME apps expect), for components too,
unless the session configuration sets it.

### Session startup (Helix)

*GNOME does not document this; Helix's choice:* gnome-session's startup
ordering (its systemd targets) is not followed. Helix's launcher starts
the components its session configuration lists, in order, waiting for
each to be ready: a component is ready once it owns a given bus name, or
once it has called `Setenv` for a given variable (the compositor sets
`WAYLAND_DISPLAY`). Then the session is initialized.

### Memory pressure (Helix)

*GNOME does not document this; Helix's choice:* how a session should
protect itself from systemd-oomd is not documented. Helix's packages
install a drop-in (`<unit>.d/50-helix-oomd-avoid.conf` under
`/usr/lib/systemd/user/`) for the user services a session cannot lose:
the session bus (`dbus.service`, `dbus-broker.service`), the
accessibility bus (`at-spi-dbus-bus.service`) and the portal services
(`xdg-desktop-portal.service`, `xdg-document-portal.service`,
`xdg-permission-store.service`). It sets:

- `ManagedOOMPreference=avoid`: when the user manager
  (`user@UID.service`) is under memory pressure, systemd-oomd chooses
  other units (applications, builds, other services) first. systemd
  honours the preference for user units because the unit's cgroup and the
  monitored `user@UID.service` belong to the same user.
- `OOMScoreAdjust=100`: the user manager's own level, below the 200 that
  user services get by default, so the kernel's OOM killer also prefers
  other processes of the user.

The compositor, shell, portals backend and other session components are
children of `helix-session`, which runs in the login's session scope
(`session-N.scope`, created by logind) rather than under the user
manager; systemd-oomd's pressure handling for `user@UID.service` does
not reach them. Losing the session bus ends the session (see "When the
bus goes away"), which is why the bus is protected.

Apps the shell starts (from the launcher, app grid, dock, dash, launch
keys, notifications, the run dialog, the welcome tour, the background
menu and extensions)
each get a transient scope of their own under the user manager:
`app-helix-<app id>-<pid>.scope` in `app.slice`, following systemd's
naming for desktop environments (the app id is the desktop file id
without `.desktop`, escaped as systemd escapes unit names, so `-` is
`\x2d`). Right after GIO spawns the app, the shell calls
`org.freedesktop.systemd1.Manager.StartTransientUnit` on the session bus
(name, mode `fail`, properties `Description` "Application launched by
helix-shell", `PIDs` the app's pid, `Slice` `app.slice`, `CollectMode`
`inactive-or-failed`). This moves the app out of the session's scope
into `user@UID.service`, where systemd-oomd's pressure handling sees it,
so under memory pressure oomd has each app as a candidate of its own
rather than the protected services. Apps started by D-Bus activation are
started by the bus and are not moved. Without a user manager on the bus
the call fails and the app stays where it was started.
*GNOME does the same for apps it launches (`app-gnome-…`); the exact
properties are Helix's choice.*

Checking this by hand (it needs a machine or VM whose session may be
logged out; not covered by automated tests):

1. In a Helix session, open an app from the launcher and run
   `systemctl --user list-units 'app-helix-*'`: it lists the app's scope,
   and `systemctl --user status <scope>` shows it in `app.slice`.
2. `systemctl --user show dbus.service -p ManagedOOMPreference` reports
   `avoid`; `oomctl` lists `user@UID.service` among the cgroups monitored
   for memory pressure.
3. Start a memory hog from the launcher (for example an app entry running
   `stress-ng --vm 1 --vm-bytes 95% --vm-keep`, or `tail /dev/zero` in a
   terminal app), wait for pressure to pass the limit (`oomctl`), and
   read `journalctl -u systemd-oomd`: oomd kills the hog's
   `app-helix-*.scope` (or another app scope), the session bus stays, and
   the session goes on.

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
- `app_id` must be a plain desktop file id: ASCII letters, digits, `_`,
  `-` and `.`, with no empty element (no leading or trailing dot, no
  `..`) and at most 255 bytes. Others fail with
  `org.gnome.SessionManager.InvalidArgs`, and saved records whose id is
  not one are dropped when the file is read.
- `dbus_name` must be the caller's own bus name (its unique name, or a
  name it owns); empty means its unique name. Otherwise the call fails
  with `org.gnome.SessionManager.InvalidArgs`. The instance counts as
  gone when that name leaves the bus.
- An instance is restorable when it was still registered at the end of
  the session (reason 3), or when its app left the bus without
  unregistering, or the whole session ended without the session manager
  ending it (reason 2). Each restorable instance is handed out once, to
  the next `RegisterRestore` of its app; otherwise the app gets a new id
  and reason 1.
- At login, once the session's components are up, every app with
  restorable instances is started once per instance, through its desktop
  file (`<app_id>.desktop` in the `applications` directory of the XDG
  data directories).
- Unregistered instances become cleanup ids, returned to the app's next
  `RegisterRestore` until it reports them deleted.
- `org.gnome.desktop.session` `save-restore` (true without the key):
  when false, no apps are started at login, instances still registered
  when the session ends are given up instead of saved, and a
  `RegisterRestore` gives up its app's instances left from earlier
  sessions (they become cleanup ids; the app gets a new id and reason 1).
  The value is read at each of these moments. Whether GNOME also skips
  restoring after a crash when it is off is not documented; Helix does.
- *GNOME does not document this; Helix's choice:* how the reason is
  decided, when apps are relaunched and where records are kept, as
  above. Whether xdg-desktop-portal uses this API under
  `XDG_CURRENT_DESKTOP=Helix:GNOME` has not been tried.
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

*GNOME does not document this; Helix's choice:* gnome-session's autostart
phases, waits, delays, restart rules and full list of conditions; Helix's
are below.

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

*GNOME does not document this; Helix's choice:* when a caller's bus
connection goes away, its clients and inhibitors are dropped (so a
crashed app cannot block logout forever).

## Errors

*GNOME does not document this; Helix's choice:* errors are named under
`org.gnome.SessionManager.Error`:

| Name | When |
|------|------|
| `NotInInitialization` | `Setenv` or `InitializationError` after the initialization phase |
| `AlreadyInitialized` | the session is initialized a second time |
| `NotInRunning` | ending the session before it is running |
| `NotRegistered` | an unknown client |
| `InvalidCookie` | an unknown inhibit cookie |
| `InvalidArgs` | an unknown `Logout` mode |
| `Inhibited` | ending the session is blocked by an inhibitor |
| `General` | anything else (no free inhibit cookie, the session already ending) |

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
it.

*GNOME does not document this; Helix's choice:* how gnome-session (and
systemd) behave in these cases is not documented; Helix's is above.

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

## `SetRebootToFirmwareSetup(b enable)`, `CanRebootToFirmwareSetup`

`SetRebootToFirmwareSetup` asks that the next restart open the
firmware's setup screen (`enable` true), or no longer (false). It does
not restart by itself; the user then restarts as usual (`Reboot`, or the
end-session dialog). `CanRebootToFirmwareSetup` tells a UI whether to
offer this.

### Helix

Both go to logind's methods of the same names. `CanRebootToFirmwareSetup`
is true when logind answers `yes` or `challenge` (the request may then
need authentication, which logind arranges), false otherwise and
without logind. `SetRebootToFirmwareSetup` returns logind's refusal as
an error, and fails when there is no logind.

*GNOME does not document this; Helix's choice:* whether `challenge`
counts as available is not documented; Helix offers the action and lets
logind ask for authentication.

## `SessionIsActive`

Follows the `Active` property of the logind session the session manager
runs in (`GetSession($XDG_SESSION_ID)`, or `GetSessionByPID` of its own
process when the variable is unset), with `PropertiesChanged` on every
change. Without logind it is true.
