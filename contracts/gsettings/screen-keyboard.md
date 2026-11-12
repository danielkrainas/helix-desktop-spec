# On-screen keyboard (Helix)

The shell's own on-screen keyboard, for
`org.gnome.desktop.a11y.applications screen-keyboard-enabled`
([key table](org.gnome.desktop.a11y.applications.md)); the
`on-screen-keyboard` key flips that setting
(`a11y-magnifier.md`, "Keys").

## When it shows

While the setting is on and an app has a text field focused (the shell
is the session's input method, `zwp_input_method_v2`: between `activate`
and `deactivate`), the keyboard is shown along the bottom of the primary
monitor, above windows; it reserves its height (a layer-shell exclusive
zone) so windows are laid out above it. It hides when the text field
loses focus or the setting goes off. Its "Hide" key hides it until a text
field is focused again. Clicking it never moves keyboard focus.

When another program is the input method, the shell's keyboard does not
show (it needs the input method to type).

## Layout

Five rows, built from the active input source's layout (an `xkb` source's
layout and variant; `us` for other sources) by compiling it with the
`evdev` rules: for the main block's keys (evdev codes 41, 2–13; 16–27,
43; 30–40; 86 when its characters are on no other key, 44–53), the characters of their
first and second (Shift) levels. Keys whose levels give no character
(e.g. dead keys) are left out. Around them:

- row 1 ends with **Backspace**; row 2 starts with **Tab**; row 3 ends
  with **Enter**; row 4 has **Shift** at both ends;
- row 5 (on every page): **Hide**, **?123** (on the symbols page:
  **ABC**), **Emoji** (on the emoji page: **ABC**), **Space**, **←**,
  **→**.

## Pages

The letters page above, and two more; each keeps row 5 and ends its
first row with Backspace:

- **Symbols**: `1234567890`; Tab, `@#$%&*-+()/`; `!"':;,.?_=` and Enter;
  `~` `` ` `` `|` `\` `<>` `[]` `{}` `^` `€`.
- **Emoji**: three rows of ten: 😀 😂 😊 😍 😘 😎 🤔 😢 😭 😡 / 👍 👎 👏 🙏
  💪 👋 🤝 ✌ 👌 🙌 / ❤ 💔 ⭐ 🔥 🎉 ✅ ❌ ⚠ 💡 ☕. They are drawn with the
  font fontconfig gives for `emoji:color=false` (a font with outlines;
  color emoji fonts are not drawn), and type the character.

The keyboard opens on the letters page; typing a character does not
change the page.

## Typing

- A character key types its character (its Shift level while Shift is
  on) as text committed by the input method; Shift then goes off.
- Shift toggles the Shift level for the next character.
- Backspace, Enter, Tab, ← and → press and release those keys (evdev 14,
  28, 15, 105, 106) on the input method's virtual keyboard
  (`zwp_virtual_keyboard_v1`); when it has no keymap yet, the shell gives
  it the `us` keymap first.
