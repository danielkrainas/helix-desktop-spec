# Power management (Helix)

What Helix does that gnome-settings-daemon's power plugin does in GNOME,
following `org.gnome.settings-daemon.plugins.power`
(`org.gnome.settings-daemon.plugins.power.md`), `org.gnome.desktop.session`
`idle-delay` and UPower. The shell does it; changes apply at once.

"Idle" is the session idle time (`org.gnome.Mutter.IdleMonitor`); nothing
below happens while an app inhibits idleness (`org.gnome.SessionManager`
`Inhibit` flag 8), and any input undoes dimming and blanking at once.

## Dimming and blanking

- With `idle-dim` true and a controllable built-in backlight
  (DisplayConfig `Backlight`), the panel dims after four fifths of
  `idle-delay` (at least 10 seconds before blanking): to `idle-brightness`
  percent, or not at all when it is already that dark. Activity brings
  the previous brightness back.
- After `idle-delay` seconds (0: never) the monitors turn off: DisplayConfig
  `PowerSaveMode` 3. Activity turns them on (`PowerSaveMode` 0).
- Locking after `lock-delay` is unchanged
  (`org.gnome.desktop.screensaver`, see `org.gnome.ScreenSaver.md`).

## Automatic suspend

On mains power (UPower `OnBattery` false, or no UPower) after
`sleep-inactive-ac-timeout` seconds idle, on battery after
`sleep-inactive-battery-timeout` (0: never), the shell does
`sleep-inactive-ac-type` / `-battery-type`:

- `suspend`: `org.gnome.SessionManager` `Suspend`;
- `hibernate`: logind `Hibernate`, when `CanHibernate` is `yes` and the
  session manager's `CanSuspend` is not 0 (a session that may not suspend
  may not hibernate either);
- `shutdown`: `Shutdown` (the end-session dialog, which counts down);
- `logout`: `Logout(1)` (no confirmation);
- `interactive`: the power-off end-session dialog;
- `blank`: the monitors turn off (as above);
- `nothing`.

It happens once per idle period. A suspend inhibitor (flag 4) holds back
`suspend` and `hibernate`.

## Power button and lid

- Only in a session whose `CanShutdown` is not 0 (one that powers the
  machine off through logind), the shell takes logind's `handle-power-key` lock (`Inhibit("handle-power-key",
  …, "block")`), so logind leaves the power button to it. The button
  (`XF86PowerOff`, the `power` media key) does `power-button-action`:
  `suspend`, `hibernate` (as above), `interactive` (the power-off dialog) or
  `nothing`.
- While an external monitor is connected (DisplayConfig
  `HasExternalMonitor`), the shell holds logind's `handle-lid-switch`
  lock, so closing the lid does not suspend.
- The compositor follows UPower's `LidIsClosed` (`org.freedesktop.UPower`
  on the system bus, while `LidIsPresent`). When the lid closes and a
  monitor other than the built-in one (DisplayConfig `is-builtin`) is on,
  it applies the current configuration without the built-in monitors, as
  a temporary change: the remaining logical monitors are moved so the
  layout starts at 0,0, and the first of them becomes primary when the
  built-in one was. When the lid opens, the configuration from before is
  applied again if the same monitors are still connected; otherwise the
  built-in monitor stays off until the user changes the configuration.
  Monitors plugged in while the lid is closed are handled as usual.

## Battery

From UPower's display device `WarningLevel`:

- 3 (low): a notification "Battery low" ("<n> % remaining"); with
  `power-saver-profile-on-low-battery` the power profile becomes
  `power-saver` (the previous one comes back when the battery charges or
  is no longer low, unless it was changed meanwhile);
- 4 (critical): "Battery critically low" ("Plug in soon…"), urgent;
- the critical action itself is UPower's.

## Ambient light

With `ambient-enabled` on, on bare hardware, when iio-sensor-proxy
(`net.hadess.SensorProxy` at `/net/hadess/SensorProxy`, system bus)
reports `HasAmbientLight`, the shell claims the sensor (`ClaimLight`;
`ReleaseLight` when the key goes off) and follows `LightLevel`:

- The wanted brightness is 10 % + 90 % × log10(1 + lux) / log10(2001)
  (full at 2000 lux and above; with `LightLevelUnit` `vendor` the level
  is taken as 0–100 % of the range instead), plus the user's offset,
  kept within 1–100 %.
- Twice a second the backlight (every DisplayConfig `Backlight` entry, as
  the brightness keys set it) moves a third of the way there (at least 1
  %); within 2 % it is left alone.
- A brightness change made by someone else (the keys, Settings) is taken
  as the user's choice: the offset becomes that brightness less the
  curve's value for the current light, so it stays and moves with the
  light from there.
- Nothing changes while the screen is dimmed for idleness.

How gnome-settings-daemon turns light into brightness is not documented
here; this curve is Helix's. Not tried with a real sensor.
