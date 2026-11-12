# Keyboard accessibility (Helix)

How Helix applies `org.gnome.desktop.a11y.keyboard`
(`org.gnome.desktop.a11y.keyboard.md`). The compositor applies these to
keys from input devices before shortcuts and apps see them; keys injected
by remote desktop sessions are not affected. Changes apply at once.

"Beep" below plays the sound theme's `bell` sound.

## Sticky keys (`stickykeys-enable`)

- Modifiers are Shift, Control, Alt (and AltGr) and Super, left or right.
- A modifier pressed and released with no other key in between is
  *latched*: it applies to the next other key, and is let go when that key
  is released. Doing the same again while it is latched *locks* it: it
  applies to every key until it is pressed and released alone a third
  time, which unlocks it.
- Holding a modifier down while pressing other keys works as usual.
- `stickykeys-two-key-off`: pressing another key while a modifier is held
  down turns sticky keys off (`stickykeys-enable` becomes false).
- `stickykeys-modifier-beep`: beep whenever a modifier is latched, locked
  or unlocked.

## Slow keys (`slowkeys-enable`)

- A key counts as pressed only once it has been held for `slowkeys-delay`
  milliseconds; then apps get the press (and later the release). A key
  released earlier is ignored.
- Beeps: `slowkeys-beep-press` when a key goes down, `slowkeys-beep-accept`
  when it is accepted, `slowkeys-beep-reject` when it is ignored.

## Bounce keys (`bouncekeys-enable`)

- A key pressed again less than `bouncekeys-delay` milliseconds after it
  was released is ignored (its press and its release).
  `bouncekeys-beep-reject`: beep when that happens.

## Toggle keys (`togglekeys-enable`)

- Beep when Caps Lock, Num Lock or Scroll Lock is pressed.

## Mouse keys (`mousekeys-enable`)

While Num Lock is off, the keypad drives the pointer instead of typing:

- 1, 2, 3, 4, 6, 7, 8, 9 move it in their direction (diagonals with the
  corner keys): one pixel at once, then, after `mousekeys-init-delay`
  milliseconds held, continuously, speeding up evenly to
  `mousekeys-max-speed` pixels per second over `mousekeys-accel-time`
  milliseconds. Several held keys add up.
- `/`, `*` and `-` choose the button: left (the default), middle, right.
- 5 clicks the chosen button, `+` double-clicks it, 0 presses it (and
  holds it) and `.` releases it.

## Turning features on and off from the keyboard (`enable`)

When `enable` is true:

- pressing Shift five times in a row (no other key in between, at most
  10 seconds apart from first to last) turns sticky keys on or off;
- holding Shift for 8 seconds turns slow keys on or off.

`feature-state-change-beep`: beep when either happens, and when the
timeout below turns features off.

## Timeout (`timeout-enable`)

After `disable-timeout` seconds with no key pressed, sticky, slow, bounce,
toggle and mouse keys are turned off (their `-enable` keys become false).

## Not supported

`org.gnome.desktop.a11y.mouse` (simulated secondary click and dwell
click) and `org.gnome.desktop.a11y` `always-show-universal-access-status`
(Helix has no accessibility menu in the panel).
