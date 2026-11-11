# Night Light — how Helix applies the settings

Behavior for the `night-light-*` keys of
`org.gnome.settings-daemon.plugins.color` (key table in the file named
after the schema). In GNOME, gnome-settings-daemon runs Night Light; in
Helix the shell does, through DisplayConfig's `SetCrtcGamma`
(`contracts/dbus/org.gnome.Mutter.DisplayConfig.md`). Changes apply at
once.

## When it is on

Night Light is on while `night-light-enabled` is true and the current
local time is inside the schedule:

- With `night-light-schedule-automatic` true and usable
  `night-light-last-coordinates` (latitude −90…90, longitude −180…180;
  the default `(91, 181)` means unknown), from sunset to sunrise at those
  coordinates (the sun's centre 0.833° below the horizon, the standard
  allowance for refraction and the sun's size). Where the sun does not set
  that day it is off; where it does not rise, on.
- Otherwise from `night-light-schedule-from` to `night-light-schedule-to`,
  in hours after local midnight (20.5 is 20:30). When "from" is later than
  "to", the schedule runs over midnight. Equal times mean never.

Helix does not look up the location itself (GNOME uses GeoClue to keep
`night-light-last-coordinates` current); it uses the coordinates stored
there.

## What it does

While on, every monitor's colors are warmed to
`night-light-temperature` (kelvin, clamped to 1000…10000); while off they
are left at 6500 K, which changes nothing. Changes fade over 5 seconds.
The schedule is checked every minute, and the gamma is set again when
monitors change.

The white point for a temperature is the blackbody color approximated
per channel with Tanner Helland's published fit, divided by its value at
6500 K so that 6500 K gives (1, 1, 1). Each CRTC gets linear ramps from 0
to that factor (times 65535), in the size `GetCrtcGamma` reports.

Its state is published as `org.gnome.SettingsDaemon.Color`
(`contracts/dbus/org.gnome.SettingsDaemon.Color.md`), which can also
preview it and disable it until tomorrow.
