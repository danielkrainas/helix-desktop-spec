# Message tray calendar (Helix)

The list that opens from the clock (`contracts/dbus/org.freedesktop.Notifications.md`,
"The notification list") has a second column, right of the notifications:
the date, a month calendar, the chosen day's events, world clocks and the
media player. It is read when the tray opens and after changes made from
it (another month, a media button); the clock times follow the minute.

## Date and month

- The weekday and the full date of today.
- A month grid, starting with today's month: its name and year, with
  buttons for the previous and next month; a row of weekday initials; six
  weeks of day numbers, days of other months dimmed.
- The week starts on the locale's first weekday (glibc's `LC_TIME`
  `first_weekday`, counted from `week-1stday`); Sunday when unknown.
- With `org.gnome.desktop.calendar show-weekdate` true, a column of ISO
  week numbers comes first.
- Today is marked; clicking a day chooses it (today at first). Days with
  events have a dot.

## Events

From Evolution Data Server (`contracts/dbus/org.gnome.evolution.dataserver.md`):
the events of every enabled, selected calendar for the six weeks shown.
Under the grid, a heading for the chosen day ("Today", "Tomorrow", or its
weekday and date), then that day's events ordered by start: "All Day" or
the start time (24-hour), and the summary; at most six, then "N more".
"No Events" when there are none.

An event is on a day when it overlaps it in local time. Recurring events
are expanded for `FREQ` `DAILY`, `WEEKLY` (with `BYDAY` weekdays),
`MONTHLY` and `YEARLY`, with `INTERVAL`, `COUNT` and `UNTIL`; `EXDATE`s and
detached instances (`RECURRENCE-ID`) replace occurrences; other rule parts
are not used (see `gaps.md`). Cancelled events (`STATUS:CANCELLED`) are
left out.

## World clocks

`org.gnome.shell.world-clocks locations` (`av`): each value is a
serialized GWeather location, `(uv)` with version 2 and
`(ssba(dd)a(dd))`: name, weather station code, whether it is a city, the
city's coordinates and the station's, as (latitude, longitude) in
radians. Under a "World Clocks" heading, each location's name and its
current time (24-hour), with the weekday when it differs from today's
here. The time zone is the one whose principal city in the system's
`zone1970.tab` is nearest to the location (see `gaps.md`).

## Media

The player the playback keys control (`contracts/dbus/org.mpris.MediaPlayer2.Player.md`),
when there is one: its name (`Identity`), the track's title and artists
(`Metadata` `xesam:title`, `xesam:artist`), and buttons for previous,
play/pause (showing pause while playing) and next.
