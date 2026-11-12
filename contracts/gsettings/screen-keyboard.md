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
  **ABC**), **AltGr** (only for layouts that have an AltGr page, below;
  on that page: **ABC**), **Emoji** (on the emoji page: **ABC**),
  **Space**, **←**, **→**.

Above the rows is the suggestion bar (below), 40 pixels high, always
reserved so the keyboard's height does not change.

## Pages

The letters page above, and two more; each keeps row 5 and ends its
first row with Backspace:

- **Symbols**: `1234567890`; Tab, `@#$%&*-+()/`; `!"':;,.?_=` and Enter;
  `~` `` ` `` `|` `\` `<>` `[]` `{}` `^`, then the layout's currency
  sign and, when that is not `€`, `€`. The currency sign by layout:
  `$` (us, ca, au, latam), `R$` (br), `£` (gb), `¥` (jp, cn), `₹` (in),
  `₽` (ru, by), `₴` (ua), `₩` (kr), `₺` (tr), `₪` (il), `zł` (pl),
  `Kč` (cz), `CHF` (ch), `kr` (se, no, dk, is), `₫` (vn), `฿` (th),
  `€` for every other layout.
- **AltGr**: the main block's third-level characters (fourth with
  Shift), row by row as on the letters page, from the same compiled
  layout; a key whose third level repeats its first, repeats an earlier
  key of the row, or is a character the keyboard's font cannot draw is
  left out. Row 1 ends with Backspace, row 3 with Enter, row 4 starts
  with Shift. Layouts with fewer than four such characters (such as
  `us`) have no AltGr page.
- **Emoji**: three rows of ten: 😀 😂 😊 😍 😘 😎 🤔 😢 😭 😡 / 👍 👎 👏 🙏
  💪 👋 🤝 ✌ 👌 🙌 / ❤ 💔 ⭐ 🔥 🎉 ✅ ❌ ⚠ 💡 ☕. They are drawn with the
  font fontconfig gives for `emoji:color=false` (a font with outlines;
  color emoji fonts are not drawn), and type the character.

The keyboard opens on the letters page; typing a character does not
change the page.

## Typing

- A character key types its character (its Shift level while Shift is
  on) as text committed by the input method when it is let go (see
  "Accents" for a long press); Shift then goes off.
- Shift toggles the Shift level for the next character.
- Backspace, Enter, Tab, ← and → press and release those keys (evdev 14,
  28, 15, 105, 106) on the input method's virtual keyboard
  (`zwp_virtual_keyboard_v1`); when it has no keymap yet, the shell gives
  it the `us` keymap first.

## Accents

Holding a letter key for 450 ms or more and letting go offers its
accented forms in the suggestion bar instead of typing it; clicking one
types it, and any other key puts the suggestions back. A short press
types the letter when it is let go (on the key it was pressed on).

| Letter | Offered |
|--------|---------|
| a | à á â ä ã å ā ą æ |
| c | ç ć č |
| d | ď ð |
| e | è é ê ë ē ę ě |
| g | ğ |
| i | ì í î ï ī ı |
| l | ł ľ |
| n | ñ ń ň |
| o | ò ó ô ö õ ø ō ő œ |
| r | ř |
| s | ß ś š ş |
| t | ť ţ þ |
| u | ù ú û ü ū ů ű |
| y | ý ÿ |
| z | ź ż ž |

With Shift on (or for a capital letter key) they are offered as
capitals, except `ß` and `ı`.

## Languages

The keyboard's language follows the active input source's layout:
`us`, `gb`, `au`, `ie`, `nz`, `za` → English; `de`, `at`, `ch` →
German; `fr`, `be`, `ca` → French; `es`, `latam` → Spanish; `it`;
`pt`, `br` → Portuguese; `nl`; `se` → Swedish; `no` → Norwegian
Bokmål; `dk` → Danish; `fi`; `is`; `pl`; `cz` → Czech; `sk`; `hu`;
`ro`; `si` → Slovenian; `hr`; `rs` → Serbian; `bg`; `ru`; `ua` →
Ukrainian; `by` → Belarusian; `gr` → Greek; `tr`; `ee` → Estonian;
`lv`; `lt`; `il` → Hebrew; `ara` → Arabic; `ir` → Persian; `in` →
Hindi; `jp`, `kr`, `cn`, `vn`, `th`; anything else → English.

Its word list is the installed hunspell dictionary for that language
(`<language>_<REGION>.dic` or `<language>.dic` in `/usr/share/hunspell`,
`/usr/share/myspell` or `/usr/share/myspell/dicts`), preferring the
region of the session's locale (`LC_ALL`, `LC_MESSAGES`, then `LANG`),
else the first by name. Each line after the first count line is a word,
up to its `/` flags; words of one letter, or with characters other than
letters, `'` and `-`, are not used.

## Suggestions

While the letters page is shown, the suggestion bar offers up to three
completions of the word before the cursor:

- **The word before the cursor** is the run of letters (and `'`) right
  before the cursor in the text the text field reports
  (`zwp_input_method_v2.surrounding_text`); for fields that do not
  report it, the letters typed on the keyboard since the word was last
  ended (Space, Enter, Tab, ←, →, a non-letter character, a page change
  or a picked suggestion end it; Backspace takes back one).
- **Completions** are words starting with it (ignoring case), not the
  word itself: first words typed this session (most typed first), then
  the word list's, shortest first. They take the case of what was typed
  (a first capital, or all capitals for two or more capitals).
- **Picking one** deletes the word before the cursor
  (`delete_surrounding_text`) and types the completion followed by a
  space.
- **Words typed this session** are remembered when a word is ended or a
  suggestion is picked, only in memory: they are never written to disk
  and are forgotten when the shell ends.
- **No suggestions, and nothing remembered,** in fields whose content
  purpose (`zwp_text_input_v3`) is digits, number, phone, password, PIN,
  date, time, date and time or terminal, or whose hints include hidden
  text or sensitive data.
- Without a word list for the language, only words typed this session
  are offered.

