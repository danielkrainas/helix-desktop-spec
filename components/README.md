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
