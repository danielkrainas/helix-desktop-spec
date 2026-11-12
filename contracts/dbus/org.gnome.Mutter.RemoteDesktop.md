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
- Clipboard and keymap methods, like input, need a started session
  (else `org.freedesktop.DBus.Error.Failed`).
- EIS (`ConnectToEIS`, after `Start`): see below.

### Clipboard

The session shares the clipboard (the `CLIPBOARD` selection of Wayland
and X11 apps; not the primary selection).

- `EnableClipboard(options)` turns sharing on; again while on fails with
  `org.freedesktop.DBus.Error.Failed`. A non-empty `mime-types` (`as`)
  also does `SetSelection`. If the clipboard has content then (not the
  session's own), `SelectionOwnerChanged` is emitted at once. While
  sharing is off, the other clipboard methods fail with
  `org.freedesktop.DBus.Error.Failed`.
- `SelectionOwnerChanged(options)` is emitted whenever the clipboard
  changes: `mime-types` (`as`, empty when the clipboard was cleared) and
  `session-is-owner` (`b`, true when this session set it).
- `SetSelection(options)`: the session's clipboard content, offered in
  `mime-types` (`as`; missing or empty:
  `org.freedesktop.DBus.Error.InvalidArgs`), becomes the clipboard.
- When an app pastes one of those types, `SelectionTransfer(mime_type,
  serial)` is emitted with a new serial. `SelectionWrite(serial)` returns
  the file descriptor to write the data to (closing it ends the data);
  `SelectionWriteDone(serial, success)` finishes the transfer. An unknown
  or finished serial fails with `org.freedesktop.DBus.Error.InvalidArgs`.
  A transfer without `SelectionWrite` within 30 seconds is dropped (the
  app gets no data).
- `SelectionRead(mime_type)` returns a file descriptor to read the
  clipboard's data in that type from. A type the clipboard does not offer
  fails with `org.freedesktop.DBus.Error.InvalidArgs`; an empty clipboard,
  or the session's own content, with `org.freedesktop.DBus.Error.Failed`.
- `DisableClipboard`, or the session ending, turns sharing off: content
  the session set is cleared if it still is the clipboard, and transfers
  not yet written are dropped.

### Keymaps

- `KeymapCapabilities` is `{supported-keymap-types: [0],
  supported-xkb-keymap-formats: [1]}` (XKB, text format version 1), or
  `[1, 2]` when the system's XKB library (libxkbcommon 1.11 or later)
  reads text format version 2; Helix finds out once, by compiling a small
  keymap as version 2.
- `SetKeymap(options)` with `keymap-type` 0 replaces the keymap of the
  session's keyboard, for everything (apps, shortcuts,
  `NotifyKeyboardKeysym`): `xkb-keymap` (`h`, required) is read to its
  end (a final NUL is allowed); `xkb-keymap-format` (`u`, default 1), if
  given, must be one of the supported formats (a version 2 keymap is
  compiled as such and then used as the version 1 text the library
  writes out for it); `xkb-keymap-layout-index` (`u`, default 0) is the active
  layout; `lock-keymap` (`b`, default false). An unsupported type or
  format, a missing, unreadable or invalid keymap, or a layout index the
  keymap does not have, fail with `org.freedesktop.DBus.Error.InvalidArgs`.
  Without `keymap-type`, the configured input sources come back (when
  the keymap in effect is this session's).
- While a session's keymap is in effect, a change of the configured input
  sources (`org.gnome.desktop.input-sources`, also switching the active
  one) replaces it, unless it was set with `lock-keymap`; then such
  changes wait until the session resets its keymap or ends. When the
  session ends, its keymap gives way to the configured input sources.
- `SetKeymapLayoutIndex(index)` changes the active layout of the keymap
  this session set last, while that keymap is in effect (else
  `org.freedesktop.DBus.Error.Failed`); an index the keymap does not have
  fails with `org.freedesktop.DBus.Error.InvalidArgs`.
- `CurrentKeymap` is `{name, source}`: `name` (`s`) is the XKB name of the
  active layout (e.g. `English (US)`), `source` (`u`) 1 when the keymap in
  effect was set by this session, else 0. Every session's property
  changes (`PropertiesChanged`) when the keymap or active layout changes.
- EIS keyboards are replaced (device removed, a new one added and resumed)
  when the keymap changes, so they carry the keymap in effect. When only
  the active layout changes (`SetKeymapLayoutIndex`, or switching input
  sources within the same keymap) they stay and are sent
  `ei_keyboard.modifiers` with the new group (and the current depressed,
  locked and latched modifiers); a new EIS keyboard is sent `modifiers`
  right after it is added.

### EIS

- `ConnectToEIS`, after `Start`: Helix's EIS server offers one seat,
  `default`, with the capabilities of `device-types`. When the client
  binds them it gets virtual devices, resumed at once: a pointer
  (relative motion, buttons, scrolling), an absolute pointer and a
  touchscreen with one region per monitor (layout position and logical
  size, with the monitor's scale), and a keyboard with the current XKB
  keymap. Their events are injected as above. The connection is closed
  when the session ends.
