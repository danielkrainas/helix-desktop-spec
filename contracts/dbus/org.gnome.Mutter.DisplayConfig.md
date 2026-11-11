# org.gnome.Mutter.DisplayConfig — behavioral notes

Companion to `org.gnome.Mutter.DisplayConfig.xml`. Covers the read side used
by settings tools and portals, and changing the monitor configuration with
`ApplyMonitorsConfig`; see `gaps.md` for what is still missing.

## Bus name and object

- Bus name: `org.gnome.Mutter.DisplayConfig`
- Object: `/org/gnome/Mutter/DisplayConfig`

## Serial

Every configuration has a serial `u`. It changes whenever monitors change,
and `MonitorsChanged` is emitted. Methods taking a serial reject a stale one.

## `GetCurrentState`

`monitors` — `a((ssss)a(siiddada{sv})a{sv})`, one per physical monitor:

- monitor spec `(ssss)`: connector, vendor, product, serial
- modes `a(siiddada{sv})`: mode id, width, height, refresh rate (Hz),
  preferred scale, supported scales, properties:
  - `is-current` (`b`), `is-preferred` (`b`), `is-interlaced` (`b`)
- properties `a{sv}`:
  - `is-builtin` (`b`): a laptop panel or similar
  - `display-name` (`s`): human-readable name

`logical_monitors` — `a(iiduba(ssss)a{sv})`: x, y, scale, transform, primary,
the monitor specs shown in it, properties. A logical monitor showing more
than one monitor mirrors them. Monitors that are off appear in `monitors`
(no mode is current) but in no logical monitor.

Supported scales (per mode): Helix offers the scales from 1 to 4 in
steps of 0.25 (1, 1.25, 1.5, … 4) for which the mode's width and height
divided by the scale are both whole numbers and the logical area is at
least 800×480 pixels; 1 is always offered. A 1920×1080 mode, for example,
offers 1, 1.25, 1.5 and 2 (1.75 would not give whole logical pixels).
There is no switch for fractional scales: GNOME 51's `org.gnome.mutter`
schema has no `experimental-features` key, so they are always offered.
The preferred scale of every mode is 1.

At a fractional scale, apps that support the fractional-scale protocol
(`wp_fractional_scale_v1`) are told the exact scale and draw for it; other
apps are told the next whole scale (`wl_output` and `wl_surface` preferred
buffer scale) and their pixels are scaled down to fit. X11 apps draw at
scale 1 and are scaled up.
Helix's own shell surfaces (panel, launcher, switcher, dialogs, banners,
lock screen) draw at the exact scale of their monitor.

`transform` values: 0 normal, 1 90°, 2 180°, 3 270°, 4 flipped, 5 flipped
90°, 6 flipped 180°, 7 flipped 270°.

`properties` — `a{sv}`:

- `layout-mode` (`u`): 1 logical (logical monitor sizes are scaled), 2
  physical
- `supports-changing-layout-mode` (`b`)
- `global-scale-required` (`b`): all logical monitors must share one scale

## `ApplyMonitorsConfig`

`ApplyMonitorsConfig(u serial, u method, a(iiduba(ssa{sv})) logical_monitors,
a{sv} properties)` changes the monitor configuration. Only allowed when
`ApplyMonitorsConfigAllowed` is true.

`serial` must be the current one (from `GetCurrentState`); a stale serial
is rejected and nothing changes.

`method`:

| Value | Name       | Meaning                                          |
|-------|------------|--------------------------------------------------|
| 0     | verify     | check the configuration; change nothing          |
| 1     | temporary  | apply it; it is not remembered                   |
| 2     | persistent | apply it and remember it for this set of monitors |

Other values are rejected.

`logical_monitors`, one entry per logical monitor: x, y, scale, transform,
primary, and the monitors it shows as `(connector, mode id, properties)`.
Mode ids are those from `GetCurrentState`. Monitors not listed are turned
off.

`properties`: `layout-mode` (`u`, see above) may only differ from the
current layout mode when `supports-changing-layout-mode` is true.

A configuration is valid when:

- there is at least one logical monitor, each showing at least one
  monitor;
- every connector is a connected monitor and appears only once, with one of
  its modes;
- the monitors of one logical monitor (mirroring) use modes of the same
  size;
- the scale is one of the supported scales of the mode and the transform
  (0–7) is one the monitor supports;
- exactly one logical monitor is primary;
- in logical layout mode, a logical monitor's size is its mode size divided
  by its scale, with width and height swapped for the 90° and 270°
  transforms (1, 3, 5, 7). The logical monitors must not overlap, must
  form one connected arrangement (each shares part of an edge with
  another), and the smallest x and the smallest y must be 0.

An invalid configuration is rejected and nothing changes. When an applied
configuration changes anything, the serial changes and `MonitorsChanged` is
emitted before the call returns.

### Keeping a persistent change

A persistent configuration is applied at once, then the user is asked
whether to keep it (Helix: through the shell's
`org.helix.DisplayChangeDialog`, with a 20-second countdown).
`ApplyMonitorsConfig` returns without waiting for the answer.

- Kept: the configuration is remembered.
- Reverted, the countdown running out, or no answer: the configuration
  from before the change comes back (with `MonitorsChanged`), and nothing
  is remembered.
- With no dialog to ask, or when the configuration is the one already
  shown, it is kept and remembered at once.
- Another `ApplyMonitorsConfig` while the question is open closes it.
  The earlier change is neither remembered nor reverted; if the new one is
  persistent the question is asked again, and reverting goes back to the
  configuration from before the first unanswered change.

Temporary configurations are never asked about.

### Remembered configurations

A persistent configuration is restored whenever the same set of monitors
is connected again (monitors are told apart by connector, vendor, product
and serial), including after a restart. A temporary one lasts until the
monitors change.

## `GetResources` (legacy)

- `crtcs` `a(uxiiiiiuaua{sv})`: id, winsys id, x, y, width, height, current
  mode index (-1 if none), current transform, supported transforms,
  properties
- `outputs` `a(uxiausauaua{sv})`: id, winsys id, current CRTC (-1 if none),
  possible CRTCs, connector name, mode indices, clone indices, properties
- `modes` `a(uxuudu)`: id, winsys id, width, height, refresh rate, flags
- `max_screen_width`, `max_screen_height`

## Properties

- `PowerSaveMode` (`i`): 0 on, 1 standby, 2 suspend, 3 off, -1 unknown
- `ApplyMonitorsConfigAllowed` (`b`): whether clients may call
  `ApplyMonitorsConfig`
- `HasExternalMonitor` (`b`): any monitor that is not built in
- `Backlight` (`(uaa{sv})`): see below

## Gamma

`GetCrtcGamma(serial, crtc)` returns the current gamma ramps of a CRTC
(an id from `GetResources`): red, green and blue, 16-bit values, one entry
per step of the hardware's ramp. `SetCrtcGamma(serial, crtc, red, green,
blue)` replaces them. Both fail with `org.freedesktop.DBus.Error.InvalidArgs`
for a serial other than the current `GetResources` serial or an unknown
CRTC; `SetCrtcGamma` also when the three ramps are not all as long as the
ones `GetCrtcGamma` returns.

### Helix

- On bare hardware the ramps go to the CRTC (DRM gamma); their length is
  the CRTC's gamma size. A ramp set stays in effect across mode changes
  and switching back from another VT, until it is set again or the
  monitor goes away.
- In the nested session the ramps are kept (256 entries, identity at
  first) but have no visible effect.

## Color transformation matrix

`SetOutputCTM(serial, output, ctm)` sets the color transformation matrix
of an output (an id from `GetResources`). It fails with
`org.freedesktop.DBus.Error.InvalidArgs` for a serial other than the
current `GetResources` serial or an unknown output. The interface does not
document what the nine values mean (see `gaps.md`).

### Helix

- The nine values are a 3×3 matrix in row-major order, each in the Linux
  kernel's `drm_color_ctm` format (S31.32 fixed point, sign and magnitude:
  bit 63 is the sign, the rest the magnitude in units of 2⁻³²), applied as
  output color = matrix × input color. The identity matrix (1.0 on the
  diagonal, `0x100000000`) undoes any change.
- On bare hardware the matrix goes to the CRTC's `CTM` property; the call
  fails with `org.freedesktop.DBus.Error.NotSupported` when the CRTC has
  none (or the monitor is off). Like gamma ramps, a matrix stays in effect
  across mode changes and switching back from another VT, until it is set
  again or the monitor goes away.
- In the nested session the matrix is accepted and kept but has no visible
  effect.

## Backlight

`Backlight` is a serial and one dictionary per monitor:

| Key | Type | Meaning |
|-----|------|---------|
| `connector` | `s` | the monitor's connector |
| `active` | `b` | whether the monitor is on |
| `min` | `i` | lowest value (optional) |
| `max` | `i` | highest value (optional) |
| `value` | `i` | current value (optional) |

An entry without `min`, `max` and `value` cannot be controlled through this
interface. The property emits `PropertiesChanged` when a value or the set
of entries changes.

`SetBacklight(serial, connector, value)` sets the brightness of that
monitor; `value` is in the entry's `min`…`max` range. It fails with
`org.freedesktop.DBus.Error.InvalidArgs` for a serial other than the
current `Backlight` serial, a connector without control, or a value out of
range.

`ChangeBacklight(serial, output, value)` is deprecated (a percentage for a
`GetResources` output id); Helix does not implement it.

### Helix

- The built-in panel (connector `eDP-*`, `LVDS-*` or `DSI-*`) is
  controlled through a kernel backlight device
  (`/sys/class/backlight/*`). When there are several, `firmware` devices
  are preferred, then `platform`, then `raw`.
- Values are the device's own units: `min` is 1 (so the panel is never
  turned fully dark this way), `max` is the device's `max_brightness`.
- The value is written through logind's `Session.SetBrightness`
  (`org.freedesktop.login1.md`) and re-read from the device, so changes
  made elsewhere show up within a few seconds.
- The serial changes only when the set of entries changes (monitors
  added, removed or reconfigured), not when a value changes.
- External monitors (DDC/CI) are not controlled.
