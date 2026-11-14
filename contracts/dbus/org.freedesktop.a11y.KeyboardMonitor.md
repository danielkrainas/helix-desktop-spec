# org.freedesktop.a11y.KeyboardMonitor — behavioral notes

Companion to `org.freedesktop.a11y.KeyboardMonitor.xml`. Served by the
compositor.

On Wayland an app only sees the keys typed into its own windows. A screen
reader (Orca) needs more: it speaks what the user types anywhere, and it
has commands of its own (its "Orca key" with other keys, keypad keys in
the desktop layout) that must reach it rather than the app in front. The
compositor sees every key, so it offers this interface to assistive
technologies.

## Bus name and object

- Bus name: `org.freedesktop.a11y.Manager`
- Object: `/org/freedesktop/a11y/Manager`

If the name is taken the compositor runs without the interface.

## Who may call it

Only the client that currently owns the well-known name
`org.gnome.Orca.KeyboardMonitor` on the session bus may call the
methods; that is the screen reader's name, and the rule GNOME uses. Any
other caller gets `org.freedesktop.DBus.Error.AccessDenied` and nothing
changes. Ownership is checked on each call; watches and grabs a client
already has stay while it is connected, even if it gives the name up.

Owning that name is easy for any program on the session bus, so the rule
keeps ordinary well-behaved apps out, not malicious ones; see `gaps.md`.

## Clients

Each caller (its unique bus name) is a client with, independently:

- a **watch** (`WatchKeyboard` / `UnwatchKeyboard`),
- a **keyboard grab** (`GrabKeyboard` / `UngrabKeyboard`),
- **key grabs** (`SetKeyGrabs`): a list of modifier keysyms and a list
  of keystrokes. Each call replaces the previous lists; empty lists remove
  them.

Calling a method twice is the same as once; ending something the client
did not start has no effect. Everything a client set up is dropped when it
leaves the bus (`NameOwnerChanged` with no new owner for its unique name).

## Which keys

The monitor sees each key press and release after the keyboard
accessibility features (`contracts/gsettings/a11y-keyboard.md`: slow,
bounce, sticky and mouse keys) and before the compositor's own shortcuts
and the focused app. Keys of an input capture session
(`org.helix.InputCapture`) are not seen. Keys typed by remote desktop
sessions and by tablet pad buttons set to keystrokes are seen like keyboard
keys. The compositor does not report key
repeats (apps repeat keys themselves).

For a key the compositor works out, from the keyboard state before the key
changes it:

- `keysym`: the keysym the key produces at the current shift level and
  layout,
- `unichar`: the Unicode character it types (0 for keys that type none),
- `state`: the XKB modifier mask in effect (pressed, latched and locked
  modifiers: Shift 1, Lock 2, Control 4, Mod1 8, Mod2 16, Mod3 32, Mod4 64,
  Mod5 128),
- `keycode`: the XKB keycode (the kernel's key code plus 8).

## What happens to a key

Checked in this order for a key press:

1. **The session is locked** (the lock screen is up): the monitor stands
   aside. The key goes to the lock screen as usual and no client hears of
   it, so no assistive technology can take or record the password.
   Releases of keys pressed before the lock are the exception (rule 6).
2. **A client has grabbed the keyboard**: the key is grabbed for every
   client with a keyboard grab. Exception: the virtual terminal switching
   keys (`XF86Switch_VT_1` … `XF86Switch_VT_12`, Ctrl+Alt+F1…F12) are still
   reported but also handled as usual, so a screen reader that stops
   responding cannot trap the user.
3. **A client's modifier is held** (its press was grabbed, rule 4): the
   key is grabbed for that client — this is "Orca key + another key".
4. **The key is a client's modifier** (its keysym is in a client's
   `modifiers`):
   - the press is grabbed for every client with that modifier;
   - but if the same modifier was pressed alone (pressed and released
     with no other key in between) and is pressed again within the
     keyboard's repeat delay (`org.gnome.desktop.peripherals.keyboard
     delay`, 500 ms by default) from the first press, this second press is
     not grabbed: it is handled as usual, so a double press of Caps Lock
     still toggles Caps Lock and a double press of Insert reaches the app.
5. **The key matches a client's keystroke** `(keysym, mask)`: the key is
   grabbed for every such client. It matches when the keysym is the key's
   keysym or the keysym of the key's first shift level in the current
   layout (so `h` matches Shift+H), and the modifier mask equals `state` with Lock (2) and Mod2
   (16, Num Lock) left out of both.
6. Otherwise the key is handled as usual.

A **grabbed** key is reported to the clients it is grabbed for and does
nothing else: the focused app does not get it, compositor shortcuts do not
run, and lock and latch keys (Caps Lock, Num Lock, Scroll Lock, the XKB
lock and latch keysyms, layout switching keys) change nothing. Plain modifiers (Shift, Ctrl, Alt,
Super) still count towards `state`, so "Orca key + Shift + H" is reported
with Shift in `state`; the app sees the modifiers as they were before the
grabbed keys.

A **release** goes the way its press went: the release of a grabbed press
is grabbed for the same clients (still connected ones) — also while
locked — and the release of a key handled as usual is handled as usual.
Apps therefore never see half a key press.

Every key reported to any client is also reported to all watching clients
(not while locked). Each concerned client gets one `KeyEvent`, addressed to
it with the message's destination; nothing is broadcast.

## Not covered

`org.freedesktop.a11y.PointerLocator` (the same object in GNOME, for
finding what is under the pointer) is not served. See `gaps.md`.
