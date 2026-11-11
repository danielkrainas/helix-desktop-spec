# org.gnome.Mutter.RemoteDesktop — behavioral notes

Companion to `org.gnome.Mutter.RemoteDesktop.xml`. The compositor's
remote desktop API: a client injects keyboard, pointer and touch input
into the session, usually together with a screen cast of it. Portal
backends (and, in GNOME, gnome-remote-desktop) use it.

## Bus name and objects

- Bus name: `org.gnome.Mutter.RemoteDesktop`
- `/org/gnome/Mutter/RemoteDesktop`: `org.gnome.Mutter.RemoteDesktop`
- `CreateSession()` returns a session object
  (`org.gnome.Mutter.RemoteDesktop.Session`).

`SupportedDeviceTypes` is a bitmask: 1 keyboard, 2 pointer, 4 touchscreen.

## Sessions

- `SessionId` (`s`) names the session. A screen cast session created with
  `org.gnome.Mutter.ScreenCast.CreateSession` and the property
  `remote-desktop-session-id` (`s`) set to it belongs to this session: it
  starts and stops with it.
- `Start` starts the session (and its screen cast); `Stop` ends it.
  `Closed` is emitted when the session ends, whether or not it was
  started; it can't be used afterwards.

## Input

- `NotifyKeyboardKeycode(keycode, state)`: an evdev key code pressed
  (`true`) or released. `NotifyKeyboardKeysym(keysym, state)`: the same by
  keysym.
- `NotifyPointerButton(button, state)`: an evdev button code.
- `NotifyPointerMotionRelative(dx, dy)`: relative motion.
  `NotifyPointerMotionAbsolute(stream, x, y)`: move to (x, y) in the
  logical coordinates of screen cast stream `stream` (its object path).
- `NotifyPointerAxis(dx, dy, flags)`: smooth scrolling, in pixels of a
  standard mouse; 10.0 counts as one discrete step for clients that want
  steps. `flags`: 1 the scroll motion finished; source 2 wheel, 4 finger,
  8 continuous (at most one; finger when none).
  `NotifyPointerAxisDiscrete(axis, steps)`: `axis` 0 vertical, 1
  horizontal.
- `NotifyTouchDown(stream, slot, x, y)`, `NotifyTouchMotion(stream, slot,
  x, y)`, `NotifyTouchUp(slot)`: touch points in the stream's logical
  coordinates.
- `ConnectToEIS(options)` returns a socket to an EIS server (the libei
  protocol): the client then sends input as a libei sender. `options`:
  `device-types` (`u`), the device types to offer (default all supported).

## Helix

- Only the owner of `org.freedesktop.impl.portal.desktop.helix` (Helix's
  portal backend, which asks the user first) may call these methods;
  others get `org.freedesktop.DBus.Error.AccessDenied`. A session ends
  when its creator leaves the bus.
- `Version` is 1; `SupportedDeviceTypes` is 7.
- Input before `Start` or after the session ended fails with
  `org.freedesktop.DBus.Error.Failed`. Injected input acts like the
  user's: it moves the real pointer, changes focus, triggers shortcuts and
  counts as user activity (`IdleMonitor`); while the screen is locked it
  goes to the lock screen. Positions are clamped to the monitors.
- An absolute position names a stream of the session's screen cast; a
  monitor or virtual monitor stream maps to that monitor, a window stream
  to the window, an area stream to the area. An unknown stream fails with
  `org.freedesktop.DBus.Error.InvalidArgs`.
- `NotifyKeyboardKeysym` presses the key that gives the keysym in the
  current layout, with Shift for second-level symbols; a keysym the
  layout has no key for is ignored.
- The screen cast session of a remote desktop session refuses its own
  `Start` and `Stop` (`org.freedesktop.DBus.Error.Failed`).
- `CapsLockState` and `NumLockState` are the current lock states.
  `KeymapCapabilities` is empty and `CurrentKeymap` is `{source: 0}`;
  `SetKeymap` and `SetKeymapLayoutIndex` fail with
  `org.freedesktop.DBus.Error.NotSupported`, as do the clipboard methods
  (`EnableClipboard`, `DisableClipboard`, `SetSelection`,
  `SelectionWrite`, `SelectionWriteDone`, `SelectionRead`); the
  clipboard signals are never emitted.
- EIS (`ConnectToEIS`, after `Start`): Helix's EIS server offers one seat,
  `default`, with the capabilities of `device-types`. When the client
  binds them it gets virtual devices, resumed at once: a pointer
  (relative motion, buttons, scrolling), an absolute pointer and a
  touchscreen with one region per monitor (layout position and logical
  size, with the monitor's scale), and a keyboard with the current XKB
  keymap. Their events are injected as above. The connection is closed
  when the session ends.
