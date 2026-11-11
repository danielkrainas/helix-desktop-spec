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

## `GetLocale(i category)`

`category` is the platform C library's `LC_*` constant (e.g. `LC_MESSAGES`).
The reply is the session manager's current locale for that category, as the C
library reports it. An unknown category is an error.

## `InitializationError(s message, b fatal)`

Only valid during the initialization phase (before `Initialized`). Reports a
failure starting a session component. If `fatal` is true the session ends.

## `RestoreSupported`

When false, clients must not use `RegisterRestore`, `DeletedInstanceIds` or
`UnregisterRestore`.

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
