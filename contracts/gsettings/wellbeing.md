# Wellbeing: breaks and screen time (Helix)

What the Helix shell does with `org.gnome.desktop.break-reminders`
([key table](org.gnome.desktop.break-reminders.md)) and
`org.gnome.desktop.screen-time-limits`
([key table](org.gnome.desktop.screen-time-limits.md)). Settings changes
apply within a second.

"Idle" is the session idle time (`org.gnome.Mutter.IdleMonitor`
`GetIdletime`); the shell looks once a second.

## Break reminders

Each kind named in `selected-breaks` (`eyesight`, `movement`; others are
ignored) keeps its own count of active time: seconds while the session is
unlocked and idle less than 5 seconds.

- **Resting**: being idle (or locked) for `duration-seconds` without a
  pause is a break: the count starts over. A movement break counts as an
  eyesight break too.
- **Upcoming**: with `notify-upcoming`, a notification 60 seconds before
  the count reaches `interval-seconds`: "Eyesight Break Soon" / "Movement
  Break Soon", body "A break starts in one minute."
- **Due**: when the count reaches `interval-seconds`, the break starts:
  - with `play-sound`, the sound theme's `alarm-clock-elapsed`;
  - with `lock-screen`, the screen locks;
  - with `fade-screen`, the screen is covered by a darkened layer showing
    "Eyesight Break" ("Look at something far away") or "Movement Break"
    ("Stand up and move around"), with `countdown` the time left, and
    buttons "Delay" and "Skip";
  - else, with `notify`, a notification with the same title and text.
- The break ends when the user has rested `duration-seconds` (input
  starts the rest over): the layer goes, with `play-sound` the sound
  theme's `complete` plays, and the count starts over.
- **Delay** hides the layer and brings the break back after
  `delay-seconds` of active time; **Skip** ends it and the count starts
  over. Escape is Delay.
- **Overdue**: when a delayed break comes back, with `notify-overdue`, a
  notification "Break Overdue" ("You have been working for N minutes
  without a break.").
- When both kinds are due, the movement break is shown.

## Screen time

- **Active time**: seconds while the session is unlocked and idle less
  than `org.gnome.desktop.session idle-delay` (when that is 0: 5 minutes).
  Days are local calendar days.
- **History**: with `history-enabled`, each day's total is kept in
  `$XDG_DATA_HOME/helix/screen-time` (lines `YYYY-MM-DD seconds`, at most
  90 days), written every minute and read at login, so a day's total
  survives logging out. Without it, the total lives only in the session.
- **Daily limit**: with `daily-limit-enabled`, when today's total reaches
  `daily-limit-seconds`, a notification "Screen Time Limit Reached" ("You
  have used the computer for H hours and M minutes today."), once a day;
  with `grayscale`, the whole screen turns gray (the compositor's
  `org.helix.ScreenEffects`, below) until the next day, or until the limit
  is turned off or raised above the total.

## `org.helix.ScreenEffects` (Helix-private)

Served by the compositor on its `org.gnome.Mutter.DisplayConfig` name, at
`/org/gnome/Mutter/DisplayConfig`: `SetGrayscale(on b)` draws every
monitor without color (on top of the magnifier's color effects) while
`on`; property `Grayscale` (`b`). XML: `contracts/dbus/org.helix.ScreenEffects.xml`.
