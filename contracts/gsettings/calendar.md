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

## Weather

With a location in `org.gnome.shell.weather locations` (keys below), a
"Weather" section after the world clocks: the first location's name, the
current temperature and conditions with an icon, then the next four hours
(hour and temperature). The forecast comes from MET Norway's public
location forecast (`https://api.met.no/weatherapi/locationforecast/2.0/compact?lat=<lat>&lon=<lon>`,
coordinates to four decimals, with a `User-Agent` naming Helix and its
project page), at most every 30 minutes per location; the request is
made only while a location is configured. Without an answer the section
says "Weather unavailable".

- Conditions come from the forecast's symbol code (its name without the
  `_day`/`_night`/`_polartwilight` ending): `clearsky` "Clear"
  (`weather-clear-symbolic`, `weather-clear-night-symbolic` at night),
  `fair` "Mostly Clear" and `partlycloudy` "Partly Cloudy"
  (`weather-few-clouds-symbolic`, `-night-` at night), `cloudy`
  "Cloudy" (`weather-overcast-symbolic`), `fog` "Fog"
  (`weather-fog-symbolic`), names containing `thunder` "Thunderstorm"
  (`weather-storm-symbolic`), `snow` "Snow" (`weather-snow-symbolic`),
  `sleet` "Sleet", `rain`/`showers` "Rain" (`weather-showers-symbolic`;
  `lightrain`, `lightrainshowers` "Light Rain"
  (`weather-showers-scattered-symbolic`)); others "Unknown".
- Temperatures are shown in whole degrees: `org.gnome.GWeather4
  temperature-unit` `centigrade` (°C), `fahrenheit` (°F), `kelvin` (K);
  `default` follows the locale's measurement system (glibc
  `LC_MEASUREMENT` `measurement`: 2, US customary, gives °F; else °C).
- `automatic-location` true: while location services are on
  (`org.gnome.system.location` `enabled`), the shell follows GeoClue at
  city accuracy (country with `max-accuracy-level` `country`), as for
  Night Light (`night-light.md`), and the weather is for the place GeoClue
  reports, named after the principal city of the nearest
  `zone1970.tab` time zone when that city is within 100 km, else
  "Current Location". Until GeoClue reports a place (or with location
  services off), the first of `locations` is used.

`org.gnome.shell.weather` (from gnome-shell; the compiled schema at hand
was gnome-shell 46's):

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `automatic-location` | `b` | `false` |  |
| `locations` | `av` | `[]` | serialized GWeather locations, as for world clocks |

## Media

The player the playback keys control (`contracts/dbus/org.mpris.MediaPlayer2.Player.md`),
when there is one: its name (`Identity`), the track's title and artists
(`Metadata` `xesam:title`, `xesam:artist`), and buttons for previous,
play/pause (showing pause while playing) and next.
