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
