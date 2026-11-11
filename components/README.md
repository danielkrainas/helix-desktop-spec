# Components

High-level breakdown of the major pieces that make up a Helix desktop session.

These are conceptual groupings for the specification. They do not dictate the final implementation structure.

## Compositor / window management

Window management, workspace handling, and display configuration. Corresponds roughly to the role played by Mutter in a GNOME session.

## Shell UI

The user-facing shell: overview, panels, notifications, system menus, and related UI. Corresponds roughly to GNOME Shell.

## Session services

Session management, idle monitoring, screen locking, and related session-level services.

## Settings

GSettings schemas, settings daemon integration, and the settings application surface.

## Portals

XDG desktop portals and related integration points expected by sandboxed and native applications.

Additional components will be added as the specification develops.
