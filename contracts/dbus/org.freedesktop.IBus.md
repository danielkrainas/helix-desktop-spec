# org.freedesktop.IBus — what Helix relies on

Companion to `org.freedesktop.IBus.xml`: the part of IBus's D-Bus API the
shell uses as the session's input method. IBus implements it; Helix calls
it on IBus's own bus (not the session bus), at the address `ibus-daemon`
listens on.

## Input contexts

- `CreateInputContext(client_name)` returns an input context object.
- `SetCapabilities(caps)`: bits 1 pre-edit text, 2 auxiliary text, 4
  lookup table, 8 focus, 32 surrounding text. With the lookup-table bit
  the client draws the candidates itself (from `UpdateLookupTable`).
- `FocusIn` / `FocusOut` tell IBus when a text field gains or loses focus;
  `Reset` drops pending composition; `SetCursorLocation(x, y, w, h)` and
  `SetSurroundingText(text, cursor, anchor)` describe the text field.
- `ProcessKeyEvent(keyval, keycode, state)` offers a key; `handled` false
  means the client passes it on. `keyval` is the X keysym, `keycode` the
  evdev key code, `state` the X modifier mask (1 Shift, 2 Lock, 4 Control,
  8 Mod1/Alt, 64 Mod4/Super) with 1<<30 set for a release.
- Signals: `CommitText(text)` (insert it), `UpdatePreeditText(text,
  cursor, visible)` / `HidePreeditText`, `UpdateLookupTable(table,
  visible)` / `HideLookupTable`, `ForwardKeyEvent(keyval, keycode, state)`
  (deliver this key to the app).
- Text values are `IBusText` structures `(s a{sv} s v)`: the type name
  `"IBusText"`, attachments, the text, attributes. Lookup tables are
  `IBusLookupTable` structures `(s a{sv} u u b b i av av)`: page size,
  cursor position, cursor visible, round, orientation, candidates
  (`IBusText`s), labels.

## Engines

`SetGlobalEngine(name)` makes `name` the engine for every input context
(GNOME uses one global engine).

## Helix

The shell starts `ibus-daemon --panel disable --address <private address>`
(under `$XDG_RUNTIME_DIR/helix/`) the first time it needs IBus, makes one
input context with capabilities 1 | 4 | 8 | 32, and asks IBus to exit
when the shell stops.
