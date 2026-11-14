# org.gnome.ScreenSaver — behavioral notes

Companion to `org.gnome.ScreenSaver.xml`. Served by the shell.

## Bus name and object

- Bus name: `org.gnome.ScreenSaver`
- Object: `/org/gnome/ScreenSaver`

## Semantics

"Active" means the screen shield is up; in Helix that is always the lock
screen.

- `Lock`: lock the session now. Returns without waiting for the lock to be
  in place; `ActiveChanged(true)` follows when it is.
- `GetActive`: whether the session is locked.
- `SetActive(true)`: same as `Lock`. `SetActive(false)` never unlocks a
  locked session (only authentication does); it has no effect.
- `GetActiveTime`: seconds since the session was locked, 0 when unlocked.
- `ActiveChanged(new_value)`: emitted on every lock and unlock.
- `WakeUpScreen`: asks for the screen to be woken (e.g. after it blanked).

`org.gnome.SessionManager`'s `SessionIsLocked` property follows
`ActiveChanged`.

## Idle locking

The session locks automatically when the user has been idle for
`org.gnome.desktop.session idle-delay` seconds (0 = never) plus
`org.gnome.desktop.screensaver lock-delay` seconds, if
`org.gnome.desktop.screensaver lock-enabled` is true. An active
`SessionManager` inhibitor with the idle flag (8) prevents it.

## Locking before suspend

The shell holds a logind `sleep` inhibitor in `delay` mode (see
`org.freedesktop.login1.md`). On `PrepareForSleep(true)` it locks if
`org.gnome.desktop.screensaver lock-enabled` is true, waits until the lock
screen is shown, then releases the inhibitor so the system can sleep. On
`PrepareForSleep(false)` it takes a new one.

## Unlocking

Unlocking requires authenticating as the session's user through PAM.

While the password is checked (PAM can be slow: network accounts,
fingerprint fallbacks), the field keeps showing as many dots as were
typed, dimmed to half, takes no typing, and shows a spinner at its right
end in place of the unlock button; "Checking…" is announced on the
accessibility bus but not drawn. When the check fails:

- The field shakes sideways once (three swings dying out over about
  0.36 s, at most 10 logical px) and its ring turns red (#E01B24) until
  a key is typed. It is emptied and keeps the keyboard.
- The message, "Incorrect password. Try again.", is drawn right under the
  field in a light red with at least 4.5:1 contrast on the background
  (and announced). "Cannot check password: …" (PAM unavailable) is shown
  the same way, without the shake.
- With `org.gnome.desktop.interface enable-animations` off or
  `org.gnome.desktop.a11y.interface reduced-motion` `reduce`, nothing
  shakes or spins: the spinner is a still ring of dots, and the red ring
  and the message remain.

## Lock screen (Helix)

The lock screen has two states:

- **Resting**, when it locks: the time, large, and the date ("Saturday,
  November 14"; see `contracts/gsettings/calendar.md`, "Clock") a little
  above the middle, the media and notification cards under them, and at
  the bottom "Press a key or click to unlock" in small light text. No
  field, no caret, no bottom buttons. The media buttons work.
- **Prompt**: the time and date move up and shrink, and the user's
  picture and name, the password field and the bottom buttons appear,
  the cards under the field.

Any key, click, tap or touch on the lock screen (other than a media
button) shows the prompt. A key that types a character also goes into
the field, so the password can be typed straight away. The prompt goes
back to resting on Escape with an empty field (and nothing asked), and
after 30 seconds without input with an empty field (not while checking).
Going back empties the field and forgets a failure message.

On the accessibility bus the resting state publishes the clock, the
date, the media player and its buttons, the notification counts, and a
focused button "Unlock" (described "Press a key or click to unlock")
which shows the prompt.

The cards show:

- **Media**: the player the playback keys control
  (`org.mpris.MediaPlayer2.Player.md`): its track's title and artists and
  buttons for previous, play/pause and next, which work while locked. It
  is read when the screen locks, every 5 seconds while locked, and after
  each button.
- **Notifications**, with `org.gnome.desktop.notifications
  show-in-lock-screen` true: for the notifications that arrived since the
  screen locked and are still in the list, one line per app, "<app>: <n>
  notification(s)", at most five apps (most notifications first, then by
  name). Apps whose own `show-in-lock-screen` is false are left out. For
  apps whose `details-in-lock-screen` is true, the app is followed by
  their two newest notifications, one line each: the summary and the
  first line of the body (markup removed, cut to fit); other apps'
  summaries and bodies are never shown. (Per-app settings:
  `org.freedesktop.Notifications.md`, "Per-application settings".)

While Caps Lock is on (the keyboard's modifier state, followed at each
change, not only at key presses) and the field takes typing, a Caps
Lock sign (an arrow up over a bar) is drawn in the field left of the
unlock button, and "Caps Lock is on" in yellow (#F8E45C) right under the
field, above any message; the words are a label announced on the
accessibility bus. With two or more input sources
(`org.gnome.desktop.input-sources sources`), the active one's short name
("en", "de"; as in the top bar's indicator) is shown in a pill in the
field; pressing it (or Return on it after Tab) switches to the next
source, all of them in turn. The usual switch shortcut is not available
while locked.

### Transitions

- Locking: the first frame of each lock surface is the background alone
  (opaque, so nothing of the session shows); the time, date and cards
  fade in over it in 0.28 s, rising 12 logical px into place. The lock
  itself is never delayed.
- Unlocking: once the password is accepted the lock screen fades out
  over its background in 0.18 s, still locked, then the session is
  unlocked (`unlock_and_destroy`) and shows. Asked to lock again during
  that fade, it stays locked.
- Resting and prompt: the time and date move and resize, the cards move,
  over 0.24 s; the picture, name, field and bottom buttons fade in once
  the cards are mostly out of their way, rising into place.
- With `org.gnome.desktop.interface enable-animations` off, all of these
  are immediate. With `org.gnome.desktop.a11y.interface reduced-motion`
  `reduce` they only fade: nothing moves, rises or shrinks.

### Monitors

Every monitor has a lock surface, but only one shows the prompt: the
picture and name, the password field (with its ring and caret), the
cards and the bottom buttons. It is the primary monitor
(`contracts/wayland/README.md`, "Monitors and windows"), unless the user
clicked (or tapped) another monitor's lock screen, which brings the
prompt there, with what was typed. The others show the blurred picture
with the time and the date, a little larger, at 42% of the height, and
nothing to press, in the resting look whatever the prompt's state. When
the monitor with the prompt goes away (unplugged,
turned off) the prompt moves to the primary monitor (or the first left)
without losing what was typed; a monitor plugged in while locked gets
its own lock surface. A Restart / Power Off question stays where it is
asked.

### Layout

Everything is in one column in the middle of the monitor, at most 380
logical px wide (16 px from the edges on narrow monitors), spaced by a
scale of 4, 8, 12, 16, 24, 32 and 48 px, in this order: the time and the
date; a gap (48 px; 32 under 800 px of height, 24 under 700); the
user's picture and name; the password field and room under it for its
message (two lines, one under 700 px); then the media and notification
cards. The picture, name and field are centered at 45% of the height
(42% under 800 px, 40% under 700) unless the clock would not fit above
them; the clock's size and the picture's (9% and 4.5% of the height,
within bounds) follow the monitor's height. Sizes are logical, so the
same holds at fractional scales. The field's place does not depend on
the cards: it does not move when media starts or notifications come.

The bottom buttons keep a band at the bottom (their size, caption and
margin) that nothing else is drawn into. The cards get the room between
the field's message and that band: when they do not fit, notification
detail lines are dropped (the last app's first), then whole apps, which
become a last line "+N more notifications"; then the media card loses
its artist and shrinks; and if even that does not fit, it is left out.

Both are cards as wide as the password field (rounded corners of 12
logical px, a dark translucent fill; black with a white 1 px border in
high contrast), all their text at least 4.5:1 against the fill:

- The media card shows the track's art (`mpris:artUrl`, a `file://` or
  `http(s)://` address, read off the UI thread with a 5 second timeout
  and at most 8 MiB, decoded and kept per address) in an 84 px rounded
  square at the left, else the player's app icon (`DesktopEntry`), else a
  generic audio icon; the title in bold and the artists under it,
  left-aligned on one line each (shortened with "…"); and round
  previous, play/pause (larger) and next buttons at its bottom right,
  each at least 44 px to press. Art that cannot be read leaves the icon.
- The notification card has one row per app: its icon (24 px), its name
  in bold and its count in a pill at the right; the detail lines it
  allows follow under the name, the summary in bold and the body's first
  line after it, shortened to fit. (On the accessibility bus each app is
  still a label "<app>: <n> notification(s)".)

