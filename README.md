<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/helixdesktop/.github/main/brand/helix-logo-vertical-dark.svg">
    <img src="https://raw.githubusercontent.com/helixdesktop/.github/main/brand/helix-logo-vertical.svg" alt="Helix" width="140">
  </picture>
</p>

# Helix

**Helix** is a Rust desktop environment that begins with compatibility against GNOME’s public contracts and is designed to evolve beyond them.

This repository (`helixdesktop/spec`) holds the living specification and contract source of truth. The implementation is in [helixdesktop/desktop](https://github.com/helixdesktop/desktop) (compositor, shell, session, portals, login screen), and Helix's own applications are in [helixdesktop/apps](https://github.com/helixdesktop/apps). Both are tested against these contracts.

## Status

**Implemented and in daily use.** The contracts here are implemented by
helix-desktop, which runs as a full desktop session on real hardware and
is used as a primary desktop. The specification keeps growing: new
features are specified here first, and every place where GNOME's
behaviour is undocumented records Helix's own choice in the contract.
`gnome-compatibility.md` summarises where Helix stands against GNOME 51.

## Target

- **GNOME version**: 51
- **Language**: Rust
- **License**: CC0-1.0

## Compatibility goals

- Major session and service D-Bus interfaces (Shell, Mutter DisplayConfig / IdleMonitor / ScreenCast, SessionManager, key portals)
- GSettings schemas used by the desktop and common tools
- Sufficient Wayland protocol usage for normal desktop operation
- Ability to host existing GTK / libadwaita applications

Full GNOME Shell extension API compatibility is explicitly out of scope (extensions rely on unstable internal APIs). Helix's own extension system is proposed in `proposals/0001-extensions.md`.

This is a clean-room effort focused on public contracts, not a fork of GNOME source code.

## Structure

- `SCOPE.md` — what “compatible” means and current boundaries
- `components/` — high-level component breakdown, the shell's shared look, and the login screen (`components/greeter.md`)
- `contracts/` — structured contract data (D-Bus XML, GSettings schemas, Wayland notes, `contracts/extensions/` for Helix's own extension API, and `contracts/agent-access/` for Agent Access)
- `gnome-compatibility.md` — summary of what Helix supports from GNOME, what it doesn't (and why), and where it differs on purpose
- `proposals/` — designs for Helix's own features (for example its extension system), before they become contracts

The specification is living: it is refined whenever implementation work discovers missing or underspecified contracts, and changes land here before (or with) the code that follows them. Missing or unclear behaviour is recorded in the relevant contract (where GNOME is undocumented, marked *GNOME does not document this; Helix's choice:*); open work is tracked as issues in helixdesktop/desktop, labelled `gap` or `needs-hardware`.
