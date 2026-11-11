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

Supported scales (per mode): Helix offers whole-number scales from 1 to 4
that divide the mode's width and height and leave a logical area of at
least 800×480 pixels; 1 is always offered.

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
- `Backlight` (`(uaa{sv})`): serial and one dictionary per backlight
