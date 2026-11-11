# org.gnome.SettingsDaemon.Color — behavioral notes

Companion to `org.gnome.SettingsDaemon.Color.xml`. Night Light's state, as
GNOME Settings and quick-settings toggles read it. gnome-settings-daemon
does not document it; what follows is Helix's behavior (see `gaps.md`).

## Bus name and object

- Bus name: `org.gnome.SettingsDaemon.Color` (in Helix the shell owns it,
  as the shell runs Night Light: `contracts/gsettings/night-light.md`)
- Object: `/org/gnome/SettingsDaemon/Color`

## Helix

- `NightLightActive` (`b`): Night Light is warming the screen now (its
  schedule says so and it is not disabled until tomorrow, or a preview is
  running).
- `Temperature` (`u`): the temperature in kelvin applied now (6500 when
  Night Light is off; while fading, the value it fades to). Writing it is
  refused (`org.freedesktop.DBus.Error.PropertyReadOnly`); the temperature
  comes from `night-light-temperature`.
- `DisabledUntilTomorrow` (`b`): while true, Night Light stays off even
  though its schedule says on. It goes back to false by itself when the
  current Night Light period ends (the schedule says off), so it is back
  the next evening. Setting it while Night Light is off has no effect
  until the next period, which it then skips.
- `Sunrise`, `Sunset` (`d`): today's sunrise and sunset in local hours
  (6.5 is 06:30) at `night-light-last-coordinates`; -1 when the location
  is unknown.
- `NightLightPreview(duration)`: show Night Light at
  `night-light-temperature` for `duration` seconds (at most 20), even if
  it is disabled, then return to what the schedule says.
- Properties emit `PropertiesChanged` when they change.
