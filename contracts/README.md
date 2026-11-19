# Contracts

Structured contract data for the Helix living specification.

This directory holds the concrete interface definitions that an implementation must satisfy to claim compatibility with the current specification.

## Layout

- `dbus/` — D-Bus interface definitions (XML)
- `gsettings/` — GSettings schema definitions
- `wayland/` — Notes on Wayland protocols and usage
- `extensions/` — Helix's own extension API
- `agent-access/` — Agent Access, a Helix extension: the `helix-agent` CLI and the shell's `org.helixdesktop.Agent1` and `org.helixdesktop.AgentAccess1` interfaces (settings in `gsettings/org.helixdesktop.agent-access.md`)

Contracts are added as they are extracted or clarified. Missing or unclear behaviour is recorded in the contract it belongs to; where GNOME does not document something, the contract says *GNOME does not document this; Helix's choice:* and states the choice. Open work is tracked as issues in helixdesktop/desktop (labels `gap`, `needs-hardware`); `gnome-compatibility.md` at the repository root is the summary.
