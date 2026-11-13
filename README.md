# Helix

**Helix** is a Rust desktop environment that begins with compatibility against GNOME’s public contracts and is designed to evolve beyond them.

This repository (`helix-desktop-spec`) holds the living specification and contract source of truth only. Implementation will live elsewhere.

## Status

Specification phase.

## Target

- **GNOME version**: 51
- **Language**: Rust
- **License**: CC0-1.0

## Compatibility goals (v0)

- Major session and service D-Bus interfaces (Shell, Mutter DisplayConfig / IdleMonitor / ScreenCast, SessionManager, key portals)
- GSettings schemas used by the desktop and common tools
- Sufficient Wayland protocol usage for normal desktop operation
- Ability to host existing GTK / libadwaita applications

Full GNOME Shell extension API compatibility is explicitly out of scope (extensions rely on unstable internal APIs). Helix's own extension system is proposed in `proposals/0001-extensions.md`.

This is a clean-room effort focused on public contracts, not a fork of GNOME source code.

## Structure

- `SCOPE.md` — what “compatible” means and current boundaries
- `components/` — high-level component breakdown
- `contracts/` — structured contract data (D-Bus XML, GSettings schemas, Wayland notes, and `contracts/extensions/` for Helix's own extension API)
- `gaps.md` — living log of missing or unclear contracts
- `proposals/` — designs for Helix's own features (for example its extension system), before they become contracts

The specification is living. It will grow and be refined as implementation work discovers additional or clarified contracts.
