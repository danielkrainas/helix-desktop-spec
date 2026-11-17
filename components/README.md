# Components

High-level breakdown of the major pieces that make up a Helix desktop session.

These are conceptual groupings for the specification. They do not dictate the final implementation structure.

## Compositor / window management

Window management, workspace handling, and display configuration. Corresponds roughly to the role played by Mutter in a GNOME session.

## Shell UI

The user-facing shell: overview, panels, notifications, system menus, and related UI. Corresponds roughly to GNOME Shell.

### Text fields

Every field the shell draws itself (lock screen password, authentication and secret prompts, the launcher's search) shows clearly when it takes typing:

- The focused field has a ring in the accent color around it. Other fields have only a faint outline.
- The focused field shows a caret where the next character goes: after the last character, or after the last dot of a hidden password. In an empty field it sits before the placeholder.
- While a password is being checked the field takes no typing, so it loses the ring and caret until the check ends.
- A password field that just failed a check has a red ring (#E01B24), focused or not, until the next key.
- The lock screen's (and login screen's) password field warns while Caps Lock is on: a sign in the field and the words "Caps Lock is on" under it (`contracts/dbus/org.gnome.ScreenSaver.md`).

The lock screen's (and the login screen's) password field is rounded (48 logical px tall, corners of 12): a translucent light fill on the blurred background (solid black in high contrast), the focus ring a 2 px anti-aliased outline outside it (white in high contrast), the faint outline 1 px. Hidden characters are 8 px dots 16 px apart (as many as fit), and the caret, 2 px wide, keeps at least 4 px from the last dot and from the placeholder. At its right end a round button with an arrow submits, as Enter does: accent-colored when something is typed, dim and inactive when the field is empty. It is a button named "Unlock" on the accessibility bus. The other fields (authentication and secret prompts, the launcher's and the overview's search) are the shell's own field (see "Shell look"): Control filled, corners of 8 (a pill for the search fields), the same accent focus ring 2 px outside and a faint 1 px outline inside the edge otherwise. The launcher's and the overview's search fields are the same one: 600 logical px wide at most and 48 tall (growing with the text scale), a pill with the search symbol at its start, the text (or the dimmed placeholder "Type to search…") at the Entry size after it, and, while something is typed, a round flat clear button with the close symbol at its end that empties the field. There is no button to show the password on the lock screen (`org.gnome.desktop.lockdown.md`).

### Shell look

The shell's surfaces share one set of colors, with steps between
neighbours large enough to tell them apart (WCAG contrast ratio, at
least 1.3:1 between a surface and what sits on it; 3:1 for the focus
ring against a popover):

| Name | Color | Use |
|------|-------|-----|
| Panel | #1E1E24, 94 % opaque | the top bar |
| Popover | #2E2E36 | menus, quick settings, the notification list, dialogs |
| Border | #4A4A55 | a popover's 1 px edge, separators |
| Control | #45454F | a switch that is off, a button, a key, a field |
| Hovered | #5A5A64 | a control or menu row under the pointer, or selected from the keyboard |
| Pressed | #686872 | a control while pressed |
| Track | #5E5E68 | a slider's empty part (its filled part is the accent) |
| Well | #1E1E24 | a list set into a popover (networks, devices) |
| Accent | from `accent-color` (blue: #3584E4) | focus rings, a slider's filled part, indicators (not under text) |
| Accent fill | from `accent-color` (blue: #2965AF, hovered #336CB3, pressed #235594) | a switch that is on, the selected row, a suggested button: anything with text on it |

Text colors, each reaching WCAG AA (4.5:1) on the surfaces it is used
on (the panel, popovers, dialogs, wells):

| Name | Color | Use |
|------|-------|-----|
| Text | #EDEDEF | everything, also on controls and on the accent fill (except a light accent's, see "Accent color") |
| Secondary | #A4A4AD | app names, bodies, percentages, weekday initials, dialog messages, disabled entries (not on controls) |
| Error | #FF7B63 | error messages, the panel's "Sharing" and "Recording" (their dots stay #E01B24) |
| Accent text | from `accent-color` (blue: #68A3EB) | links, "Connected" |

Text never sits on the plain accent (blue's #3584E4 is 3.2:1 with Text);
buttons, selected rows and switches that are on use the accent fill, as
do the lock and login screens' suggested buttons and the round "go"
button at the end of their password field.

A hovered notification in the list is lit with a softer shade, #3A3A42,
on which Secondary text still reaches 4.5:1 (*GNOME does not document
this; Helix's choice:* rows that carry secondary text get a highlight
that keeps it readable). The screen behind a dialog is dimmed with
#0C0C10 at 69 %.

#### Shapes, space and shadows

All surfaces draw from one set of shapes (helix-draw `theme`), so they
read as one product:

| Piece | Corner radius (logical px) |
|-------|----------------------------|
| Popovers, dialogs, banners, the switcher, menus | 12 |
| Cards and pictures (window pictures, tiles) | 12 |
| Buttons, switches' tracks, fields, wells, keys | 8 |
| Rows inside a popover or list (menu entries, search results, list rows) | 6 |
| Tooltips | 6 |
| Check boxes | 4 |
| Pills (panel items, the OSD, search fields, switches, round buttons) | half their height |

Spacing comes from one scale: 4, 8, 12, 16, 24, 32, 48. Rows sit at
least 4 px inside a popover's edge, so their rounded highlight clears its
corners.

A popover has a 1 px Border edge just outside it and a soft shadow: black
at 38 % just outside its edge, fading to nothing 12 px out, the whole
shadow 2 px lower than the box. High contrast draws no shadows (its
edges are white lines). A popover that fills a surface of its own (notification
banners, the input method's candidates, the click-type and workspace
popups) has no room around it for the shadow and draws none: its Border
edge is the surface's edge and the corners outside the rounded edge stay
clear.

#### Type

The shell draws its text in the interface font: the family of
`org.gnome.desktop.interface font-name` (its first family, without style
words and size: "Cantarell 11" is Cantarell), followed while the shell
runs; without the key, or when fontconfig has nothing for the family, the
system's sans-serif font. Its bold face (fontconfig's bold of that family)
sets the panel clock, titles of dialogs, prompts and popovers, headings
(notification summaries, group and calendar headings, quick settings'
toggle titles) and bold markup. Characters the font lacks come from
fontconfig's best font for them. *GNOME does not document this; Helix's
choice:* the size in `font-name` is not used; the shell keeps its own
type scale (multiplied by `text-scaling-factor`).

With Large Text, the controls that hold text grow with it, and so do the
symbols and symbolic icons inside them, as GNOME sizes them in em: the
search fields' search and clear symbols, quick settings' toggle, slider,
list and round-button icons, and the battery icon there. App icons
(launcher rows, the switcher, the dock and app grid) keep their size.

Every size text is drawn at is one of a named scale (logical px, before
Large Text's factor):

| Name | Size | Use |
|------|------|-----|
| Tiny | 11 | counts on badges, week numbers (glanced at, never sentences) |
| Caption | 13 | secondary lines: app names and times, percentages, subtitles, hints, weekday initials; the smallest size for words to be read |
| Label | 14 | the panel, menu entries, list rows, controls' words, notification bodies |
| Body | 15 | messages, results, push buttons, list entries |
| Heading | 15, bold | notification summaries, group headers, toggle titles |
| Entry | 17 | what is typed in a search field; a folder's name |
| Subtitle | 18, bold | the calendar's date |
| Title | 20, bold | titles of dialogs, prompts and popovers |
| Large title | 30, bold | the break screen's |
| Display | 40 | a monitor's number |

Close buttons are round flat buttons with a drawn close symbol (not a
typed "×"): dim at rest, lit with Hovered under the pointer.

#### Components

Every surface builds from the same pieces, each with its looks at rest,
under the pointer (Hovered), while pressed (Pressed), on (the accent
fill), focused from the keyboard (the focus ring) and, where it can be,
unavailable (Control with its words dimmed):

- **Popover**: the frame of menus, quick settings, the notification list
  and calendar, dialogs and prompts, banners, the switcher, the
  launcher's results, the screenshot toolbar, tooltips.
- **Push button**: Control (normal), the accent fill (suggested: the
  answer most people want), red (#C01C28, hovered #C92A35, pressed
  #A51D2D: destroys or ends something), or flat (nothing at rest). Its
  label is centered at the Body size. Dialog and prompt buttons, Clear,
  banner actions, Done and Reset, Share.
- **Round button**: flat (lit only under the pointer: close buttons,
  the calendar's month arrows, small icon buttons) or filled (Control:
  media buttons, a close button over a picture).
- **Row**: a menu entry, a list row or a search result: nothing at rest,
  Hovered under the pointer or the keyboard's selection (which also has
  the focus ring inside it), Pressed while pressed, the accent fill when
  chosen.
- **Switch**: a 40×22 pill, Track when off and the accent fill when on,
  a round Text-colored knob at that end.
- **Check box**: 16–18 px with corners of 4: the accent fill with a
  check mark when checked, Control with a Secondary outline otherwise.
- **Slider**, **field**, **well** and **separator** as described above
  and under "Text fields".
- **Symbols** are drawn, not typed: the close symbol (an ×) and
  chevrons pointing left, right, up or down (submenus, month arrows, a
  group's open state), so they do not depend on the font.

In high contrast each piece keeps its shape and uses lines instead of
shades: rounded white outlines 1 px at rest and 2 px while hovered or
pressed (3 px for a pressed row), as listed below.

#### Hover and press

Everything the shell draws that can be clicked answers the pointer
before and during the click, so it is clear what is clickable and that
the click landed:

- The part under the pointer shows its hovered look (Hovered; a
  notification, the soft #3A3A42; a slider's knob, a light halo) and,
  while the button is held on it, its pressed look (Pressed; a
  notification #424249; the knob a stronger halo).
- A click acts when the button is released over the part it was pressed
  on; moving off it before releasing cancels. Sliders act from the press
  on and follow the pointer while held. A press outside a popover (quick
  settings, the notification list, the switcher, the launcher) closes it
  at once.
- Nothing stays lit once the pointer leaves the surface.
- This covers the panel's items, quick settings (its top row, sliders
  and their mute button, switches and their "›" parts, list rows, power
  actions), the notification list (notifications, group headers, close
  buttons, Clear, Do Not Disturb), the calendar (days, month arrows,
  media buttons), banners (the body, the close button, action buttons),
  the launcher's results, the switcher's entries, dialog and prompt
  buttons, check boxes, rows and "Show", the Extensions dialog and the
  panel editor, besides the dock, the app grid, menus, the overview and
  the on-screen keyboard (which have their own, under "Item feedback").
- *GNOME does not document this; Helix's choice:* the looks change at
  once, without a fade.

### Accent color

The shell and the lock screen draw with the accent the user chose in
`org.gnome.desktop.interface accent-color` (one of the nine names; an
unknown name or a schema without the key means blue), and follow it
while running: changing the key redraws every surface showing it
(panel, menus, quick settings, the notification list, dialogs, prompts,
the dock, the app grid, the overview, the lock screen, the on-screen
keyboard) without a restart. The login screen has no user settings and
takes `accent` from its own configuration (`components/greeter.md`).
The compositor's own rings (locate-pointer) stay blue.

There is one table of the nine accents in Helix: the values libadwaita
documents for its accent colors, which the Settings portal also hands to
applications (`contracts/dbus/org.freedesktop.impl.portal.md`). From
each, Helix derives the shades it draws with, checking each against the
surfaces of "Shell look" (WCAG contrast ratio):

- **Accent** (rings, indicators, a slider's fill, check marks): the
  named color, mixed toward white only as far as it takes to reach 3:1
  on a popover (and 1.5:1 on a slider's track).
- **Text on the fill**: light (Text) when the named color needs to be
  darkened by at most a quarter of the way to black for Text to reach
  4.5:1 on it; otherwise dark (#1E1E24).
- **Accent fill**: the named color moved away from its text color
  (darker under light text, lighter under dark text) until that text
  reaches 5:1. Hovered moves it up to 6/64 of the way toward white,
  pressed up to 10/64 toward black, each stopping before its text drops
  under 4.5:1.
- **Accent text** (words on the dark surfaces): the named color mixed
  at least a quarter of the way toward white, then further until it
  reaches 4.5:1 on a popover.
- A selected desktop icon's highlight is Accent at one-third opacity.

The results (Text #EDEDEF unless noted; ratios are Text on fill, Accent
on Popover, Accent text on Popover):

| Name | Accent | Accent fill (hovered, pressed) | Text on fill | Accent text |
|------|--------|-------------------------------|--------------|-------------|
| `blue` | #3584E4 (3.6) | #2965AF (#336CB3, #235594) | Text (5.0) | #68A3EB (5.1) |
| `teal` | #2190A4 (3.6) | #196E7E (#247584, #155D6A) | Text (5.0) | #59ACBB (5.2) |
| `green` | #3A944A (3.5) | #2C7139 (#367842, #255F30) | Text (5.1) | #6BAF77 (5.1) |
| `yellow` | #C88800 (4.5) | #C88800 (#CD9318, #B57B00) | #1E1E24 (5.5) | #D6A640 (6.0) |
| `orange` | #ED5B00 (3.9) | #AA4100 (#AF4D10, #8F3700) | Text (5.2) | #F28440 (5.2) |
| `red` | #E73448 (3.2) | #BE2537 (#C33647, #A01F2E) | Text (5.1) | #EE6F7D (4.6) |
| `pink` | #D56199 (3.8) | #9C4770 (#A15077, #843C5F) | Text (5.1) | #E089B3 (5.4) |
| `purple` | #A462BA (3.2) | #9141AC (#964AB0, #7A3791) | Text (5.1) | #B782C9 (4.5) |
| `slate` | #6F8396 (3.4) | #576675 (#5F6D7B, #495663) | Text (5.0) | #93A2B0 (5.2) |

High contrast ignores the accent: it keeps its own colors (blue's row
above, with white outlines), whatever `accent-color` says, so that what
it guarantees does not depend on the user's choice.

- Menus light the hovered or keyboard-selected row with Hovered.
- Quick settings' switches (pills) and round buttons are Control when off
  and Accent fill when on (their words in its text color); the Wi-Fi and Bluetooth lists sit in a Well;
  sliders show their whole Track, rounded, with a round Text-colored
  knob at the value.
- The on-screen keyboard has its own key colors
  (`contracts/gsettings/screen-keyboard.md`, "Look"). The screenshot
  toolbar (a pill) has flat icon toggles: nothing behind them when off,
  the accent fill when on.
- The keyboard focus ring is one look everywhere: a 2 logical px outline
  of the accent, 2 px outside the focused control (a gap between them),
  following the control's corner radius (a pill around a panel item, a
  rounded square around a dock item or app grid cell, a ring around a
  calendar day); white in high contrast. Where a control reaches the
  edge of its surface (a menu row, a notification banner), the ring is
  drawn just inside it instead.
- The ring shows keyboard focus only, never the pointer: it is drawn on
  what the keyboard reached (panel items with `switch-panels`, menu rows
  chosen with the arrow keys, app grid cells, dock items, quick settings
  controls, notification rows, calendar days, the focused banner, dialog
  and prompt buttons and check boxes, the overview's selected window,
  the launcher's selected result, the switcher's selected entry).
  The pointer gives a light highlight instead (menus, the launcher's
  results and the switcher's list: the Hovered row; the app grid, dock
  and the switcher's icons and tiles: a light square; the overview: a
  light frame), so after the pointer moves the keyboard's place is still
  told apart.
- *GNOME does not document this; Helix's choice:* the launcher's
  selected result and the switcher's selected entry are lit as the
  pointer's hover is (Hovered) and carry the focus ring, inside the row
  for a list row (as for a menu row), around the square for an icon or
  a tile, always (their selection is only ever the keyboard's, or the
  click that ends them); the accent fill is not used for them.
- A menu row chosen with the keyboard has both the Hovered fill and the
  ring; both go after `keyboard-focus-visible-timeout`
  (`contracts/gsettings/a11y-interface.md`), each key starting that time
  again, so focus never hides while keys are being pressed.

In high contrast (`org.gnome.desktop.a11y.interface high-contrast`) text
is white, surfaces and controls black, the accent kept (always blue's,
see "Accent color"); what tells
pieces apart is white lines rather than shades:

- Popovers and dialogs have a 2 px white edge; separators and the
  notification list's column divider are white.
- Controls that are off (switches, quick settings' round buttons, buttons, the
  calendar's month and media buttons, check boxes), list wells and
  slider tracks have a 1 px white outline; controls that are on keep the
  accent fill and the outline.
- A disabled menu entry's words are white at 60 % opacity (#999999 on
  black, still 7:1), so it is told from the enabled ones, which are
  white.
- The dock's bar and tooltip have a 1 px white outline; the item under
  the pointer a 1 px white outline around its icon (2 px while
  pressed), the keyboard's item the white focus ring outside that. App
  grid cells: 2 px while lit (3 px pressed), and the focus ring.
- The hovered or keyboard-selected menu row, launcher result or
  switcher row has a 2 px white outline, the keyboard's one the white
  focus ring inside that too; the open panel menu's button has the
  outline. The switcher's selected icon or tile has the white focus ring
  around its outlined square.
- The selected calendar day has a white ring; the Do Not Disturb switch
  an outline.
- The panel's inactive workspace dots are hollow, the active one filled
  (and wide).
- *GNOME does not document this; Helix's choice:* a quick settings
  toggle split into two parts (one with a "›") has a
  white edge on each side of the split, as wide as that part's outline.
- The screenshot toolbar's toggles that are off have the 1 px white
  outline at rest (the outline of an off control), not nothing.
- The overview: the search field is outlined in white; window pictures
  keep their colors with a 1 px white outline (3 px under the pointer);
  the workspace thumbnails are black (no wallpaper), the active one
  outlined in the accent, and a window on them that has no picture is a
  black box with a white outline (not black on black). The switcher is
  a black box with a white edge.
- These were checked in renders of every surface in high contrast, and
  the switcher and the overview also on a running (headless) session.
- The desktop's wallpaper is kept; only the lock screen's background is
  replaced (`contracts/gsettings/background.md`).

### Greeter

The login screen, run by greetd so that no GNOME component is needed to
log in: see `components/greeter.md`.

### Languages

The shell, lock screen, dialogs, portals' dialogs and the login screen
are translated and follow a locale for names and formats; right-to-left
languages mirror the shell's layouts: see `components/i18n.md`.

## Session services

Session management, idle monitoring, screen locking, and related session-level services.

## Settings

GSettings schemas, settings daemon integration, and the settings application surface.

## Portals

XDG desktop portals and related integration points expected by sandboxed and native applications.

Additional components will be added as the specification develops.
