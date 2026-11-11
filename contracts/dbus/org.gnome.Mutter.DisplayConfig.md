# org.gnome.Mutter.DisplayConfig — behavioral notes

Companion to `org.gnome.Mutter.DisplayConfig.xml`. Covers the read side used
by settings tools and portals; see `gaps.md` for what is still missing.

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
the monitor specs shown in it, properties.

`transform` values: 0 normal, 1 90°, 2 180°, 3 270°, 4 flipped, 5 flipped
90°, 6 flipped 180°, 7 flipped 270°.

`properties` — `a{sv}`:

- `layout-mode` (`u`): 1 logical (logical monitor sizes are scaled), 2
  physical
- `supports-changing-layout-mode` (`b`)
- `global-scale-required` (`b`): all logical monitors must share one scale

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
