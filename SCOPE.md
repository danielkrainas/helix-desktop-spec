# Scope

This document defines what “GNOME-compatible” means for the initial living specification of Helix.

## In scope for v0

- Major session and service D-Bus interfaces expected by a GNOME-like session:
  - Shell-related interfaces
  - Mutter DisplayConfig, IdleMonitor, ScreenCast
  - SessionManager
  - Common portals used by applications and the session
- GSettings schemas used by the desktop, settings tools, and common applications
- Wayland protocol usage sufficient for normal desktop operation (layer shell, primary selection, etc. as required by the session)
- Settings and integration points expected by existing tools and applications
- Ability to host existing GTK / libadwaita applications without modification

## Explicitly out of scope

- Full GNOME Shell extension API compatibility. Extensions patch unstable internal Shell APIs; supporting them would require continuous reverse-engineering of private interfaces.
- Exact visual, animation, or theming parity with GNOME Shell.
- Reimplementation of GTK, libadwaita, or the GObject type system.
- Complete coverage of every optional or rarely used D-Bus interface on day one.

## Living specification

The specification is living. Contracts will be added and clarified as implementation work proceeds and missing or underspecified interfaces are discovered. Gaps are tracked in `gaps.md`.

The goal is a practical compatibility surface that lets users run a Helix session and continue using their existing applications and tools, while remaining free to diverge in areas that do not break that surface.
