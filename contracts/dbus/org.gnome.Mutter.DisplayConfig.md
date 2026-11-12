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

## Monitor identity (Helix)

A monitor's vendor, product and serial (in monitor specs, and in the
`output` settings of touchscreens and tablets) come from its EDID base
block: the vendor is its three-letter PNP manufacturer id; the product is
its monitor name descriptor, else its product code as `0x` and four hex
digits; the serial is its serial number descriptor, else its serial number
as `0x` and eight hex digits. Without an EDID they are `Unknown`, `Unknown`
and `0`. The `display-name` is the product name, else the connector. The
nested (windowed) compositor's one monitor is `Helix`, `Nested`, `0`.

## `GetCurrentState`

`monitors` — `a((ssss)a(siiddada{sv})a{sv})`, one per physical monitor:

- monitor spec `(ssss)`: connector, vendor, product, serial
- modes `a(siiddada{sv})`: mode id, width, height, refresh rate (Hz),
  preferred scale, supported scales, properties:
  - `is-current` (`b`), `is-preferred` (`b`), `is-interlaced` (`b`)
- properties `a{sv}`:
  - `is-builtin` (`b`): a laptop panel or similar
  - `display-name` (`s`): human-readable name
  - `color-mode` (`u`) and `supported-color-modes` (`au`): how the monitor
    is driven, 0 default (SDR), 1 BT.2100 (HDR). Helix offers `[0, 1]`
    for monitors that can show BT.2100 (see "Color modes"), else `[0]`.
  - `is-underscanning` (`b`): present only for monitors that can
    underscan ("Adjust for TV": the picture is shrunk inside a border, for
    TVs that cut off its edges), and then whether they do. GNOME Settings
    shows its "Adjust for TV" switch for monitors that have it (observed
    with GNOME Settings 46).

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

A monitor's `properties` may hold `color-mode` (`u`), one of its
`supported-color-modes` (absent: 0); any other value makes the call fail
with `org.freedesktop.DBus.Error.InvalidArgs`. The color mode is part of
the configuration: it is applied, and remembered with persistent
changes.

A monitor's `properties` may also hold `underscanning` (`b`, absent:
false; the key GNOME Settings 46 was seen to send): true only for a
monitor that reports `is-underscanning`, else the call fails with
`org.freedesktop.DBus.Error.InvalidArgs`. It is part of the
configuration like the color mode. With the DRM backend, Helix finds the
ability in the connector's `underscan` property (an enum with `on`;
drivers offer it for some HDMI outputs) and turns it `on` with
`underscan hborder` and `underscan vborder` of 5 % of the mode's width
and height (at most 128 pixels each), or `off`; it is off when the
monitor appears. Other monitor property keys are ignored. Implemented
but not tested with real hardware: no monitor with an `underscan`
property was available; the D-Bus side is tested with the nested
session's `HELIX_TEST_UNDERSCAN` (the window then claims it can
underscan; nothing changes on screen).

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

- `PowerSaveMode` (`i`): 0 on, 1 standby, 2 suspend, 3 off, -1 unknown.
  Helix turns every monitor off (DPMS) for 1–3 and on for 0; nothing is
  drawn while they are off (the nested window shows black).
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
- A monitor's `org.gnome.mutter output-luminance` (the entry with its
  connector, vendor, product and serial and color mode 0) scales what is
  sent to its CRTC while it is in color mode 0: every ramp value is
  multiplied by the percentage / 100, taken between 10 and 100 (SDR
  cannot be brighter than full). Changes apply at once. `GetCrtcGamma`
  returns the ramps clients set, unscaled; without any, the identity
  ramp is scaled.
- A monitor's color profile calibration (see "Color profiles") applies
  after the ramps clients set and before `output-luminance`: each ramp
  value is looked up in the profile's curve for its channel.

## Color modes (Helix)

On bare hardware a monitor can be driven in **BT.2100** (color mode 1)
when its EDID's CTA-861 extension has an HDR static metadata block
listing the SMPTE ST 2084 (PQ) transfer function and a colorimetry block
listing BT.2020 RGB. In the nested session every monitor is SDR only.

When a monitor is put in BT.2100:

- its connector's `HDR_OUTPUT_METADATA` property gets the kernel's
  static metadata (type 1): EOTF ST 2084, BT.2020 primaries (red, green,
  blue) and D65 white, the EDID's desired maximum luminance (else 1000
  cd/m²) as the mastering maximum and the maximum content light level,
  its desired minimum (else 0.005 cd/m²), and its maximum frame-average
  (else half the maximum) as the maximum frame-average light level; its
  `Colorspace` property becomes `BT2020_RGB`. Back in color mode 0, the
  metadata is removed and `Colorspace` is `Default`. A monitor without
  these properties cannot change color mode (the call fails);
- each frame is composited as usual, then converted: the sRGB-encoded
  values are decoded to linear light, converted from sRGB/BT.709
  primaries to BT.2020 (the same white point), scaled so SDR white is at
  the reference luminance, and encoded with PQ. The framebuffer keeps 8
  bits per channel;
- the **reference luminance** (SDR white) is 203 cd/m² scaled by the
  monitor's `output-luminance` entry for color mode 1 (100 % is 203
  cd/m²), taken between 25 % and 300 %. Its gamma ramps are not scaled;
- monitors' `wp_color_manager_v1` descriptions change to BT.2100
  (`contracts/wayland/README.md`, "Color management");
- its color profile's calibration is not applied.

Not yet tried on an HDR monitor: the EDID parsing, the metadata, the
conversion (checked against reference values, and run on the GPU in a
nested session with `HELIX_TEST_HDR=1`, which converts the nested window
as if it were an HDR monitor) and the D-Bus side are tested without one
(`gaps.md`).

## Color profiles (Helix)

On bare hardware each monitor that is on is a display device in colord
(`org.freedesktop.ColorManager` on the system bus), so color settings can
assign it profiles:

- `CreateDevice(id, "temp", properties)` with id
  `xrandr-<vendor>-<product>-<serial>` (the monitor identity) and
  properties `Kind` `display`, `Mode` `physical`, `Colorspace` `rgb`,
  `Vendor`, `Model` (product), `Serial`, `XRANDR_name` (connector) and,
  for built-in panels, `Embedded` `true`. A monitor that goes away or is
  turned off is removed with `DeleteDevice`.
- The device's default profile is the first of its `Profiles`; the
  profile's `Filename` is read when the device is created and whenever
  colord sends `DeviceChanged` for it. When the ICC profile has a `vcgt`
  tag (a table of 8- or 16-bit entries for one or three channels, or a
  gamma/minimum/maximum formula per channel), its curves, resampled to
  the CRTC's ramp length, are the monitor's calibration (see "Gamma");
  without a profile or a `vcgt` tag there is none.
- In the BT.2100 color mode no calibration is applied.

Not yet tried with a calibrated monitor (tested against a fake colord;
`gaps.md`).

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
- External monitors are controlled over DDC/CI, on real hardware only
  (not in a nested session):
  - each active external connector's I²C bus is the `ddc` link in its
    `/sys/class/drm/card*-<connector>` directory (or an `i2c-*` entry
    there), opened as `/dev/i2c-*` at the DDC/CI address 0x37;
  - brightness is MCCS control 0x10, read with *Get VCP Feature* and
    written with *Set VCP Feature*; `min` is 0 and `max` is the maximum
    the monitor reports;
  - a monitor that does not answer, or has no brightness control, gets an
    entry without `min`, `max` and `value`; it is not asked again until it
    is plugged in again;
  - monitors are asked when they appear (within a few seconds) and
    re-read every ten seconds; a change shows up as `PropertiesChanged`;
  - setting `HELIX_NO_DDC` in the environment turns DDC/CI off.
- **Not yet tested with real monitors.** The DDC/CI code is checked only
  against a simulated monitor in Helix's tests (`gaps.md`).

## Automatic rotation (Helix)

While `org.gnome.settings-daemon.peripherals.touchscreen` `orientation-lock`
(`contracts/gsettings/org.gnome.settings-daemon.peripherals.touchscreen.md`)
is false and iio-sensor-proxy (`net.hadess.SensorProxy` at
`/net/hadess/SensorProxy`, system bus) reports `HasAccelerometer`, the
compositor claims the accelerometer (`ClaimAccelerometer`, released with
`ReleaseAccelerometer` when the lock goes on) and follows
`AccelerometerOrientation`: the built-in panel (`is-builtin`) gets the
transform `normal` 0, `left-up` 1 (90°), `bottom-up` 2 (180°), `right-up`
3 (270°); `undefined` changes nothing. The change is applied like a
temporary `ApplyMonitorsConfig` (no confirmation; `MonitorsChanged`
follows); other monitors keep their places. Without a built-in panel
nothing rotates.

