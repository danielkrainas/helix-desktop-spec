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
| Accent | #3584E4 | focus rings, a slider's filled part, indicators (not under text) |
| Accent fill | #1A64C0 (hovered #1D69C6, pressed #1856A8) | a switch that is on, the selected row, a suggested button: anything with text on it |

Text colors, each reaching WCAG AA (4.5:1) on the surfaces it is used
on (the panel, popovers, dialogs, wells):

| Name | Color | Use |
|------|-------|-----|
| Text | #EDEDEF | everything, also on controls and on the accent fill |
| Secondary | #A4A4AD | app names, bodies, percentages, weekday initials, dialog messages, disabled entries (not on controls) |
| Error | #FF7B63 | error messages, the panel's "Sharing" and "Recording" (their dots stay #E01B24) |
| Accent text | #78AEED | links, "Connected" |

Text never sits on the plain accent (#3584E4, 3.2:1 with Text); buttons,
selected rows and switches that are on use the accent fill, as do the
lock and login screens' suggested buttons.

- Menus light the hovered or keyboard-selected row with Hovered.
- Quick settings' switches and Power Mode buttons are Control when off
  and Accent fill when on; the Wi-Fi and Bluetooth lists sit in a Well;
  sliders show their whole Track.
- The on-screen keyboard's keys and the screenshot toolbar's off toggles
  are Control.
- The keyboard focus ring is 2 logical px of the accent just outside the
  focused control (a ring around a calendar day).

In high contrast (`org.gnome.desktop.a11y.interface high-contrast`) text
is white, surfaces and controls black, the accent kept; what tells
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
