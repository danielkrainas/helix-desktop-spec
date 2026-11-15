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

The lock screen's (and the login screen's) password field is rounded (48 logical px tall, corners of 12): a translucent light fill on the blurred background (solid black in high contrast), the focus ring a 2 px anti-aliased outline outside it (white in high contrast), the faint outline 1 px. Hidden characters are 8 px dots 16 px apart (as many as fit), and the caret, 2 px wide, keeps at least 4 px from the last dot and from the placeholder. At its right end a round button with an arrow submits, as Enter does: accent-colored when something is typed, dim and inactive when the field is empty. It is a button named "Unlock" on the accessibility bus. The other fields (authentication and secret prompts, the launcher's search) keep their square box. There is no button to show the password on the lock screen (`org.gnome.desktop.lockdown.md`).

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
- Quick settings' switches and Power Mode buttons are Control when off
  and Accent fill when on (their words in its text color); the Wi-Fi and Bluetooth lists sit in a Well;
  sliders show their whole Track.
- The on-screen keyboard's keys and the screenshot toolbar's off toggles
  are Control.
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
  and prompt buttons and check boxes, the overview's selected window).
  The pointer gives a light highlight instead (menus: the Hovered row;
  the app grid and dock: a light square; the overview: a light frame),
  so after the pointer moves the keyboard's place is still told apart.
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
- Controls that are off (switches, Power Mode buttons, buttons, the
  calendar's month and media buttons, check boxes), list wells and
  slider tracks have a 1 px white outline; controls that are on keep the
  accent fill and the outline.
- The hovered or keyboard-selected menu row has a 2 px white outline;
  the open panel menu's button too.
- The selected calendar day has a white ring; the Do Not Disturb switch
  an outline.
- The panel's inactive workspace dots are hollow, the active one filled
  (and wide).

### Greeter

The login screen, run by greetd so that no GNOME component is needed to
log in: see `components/greeter.md`.

## Session services

Session management, idle monitoring, screen locking, and related session-level services.

## Settings

GSettings schemas, settings daemon integration, and the settings application surface.

## Portals

XDG desktop portals and related integration points expected by sandboxed and native applications.

Additional components will be added as the specification develops.
