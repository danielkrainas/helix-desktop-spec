# Contracts

Structured contract data for the Helix living specification.

This directory holds the concrete interface definitions that an implementation must satisfy to claim compatibility with the current specification.

## Layout

- `dbus/` — D-Bus interface definitions (XML)
- `gsettings/` — GSettings schema definitions
- `wayland/` — Notes on Wayland protocols and usage

Contracts are added as they are extracted or clarified. Missing or unclear behaviour is recorded in the contract it belongs to; where GNOME does not document something, the contract says *GNOME does not document this; Helix's choice:* and states the choice. Open work is tracked as issues in helix-desktop (labels `gap`, `needs-hardware`); `gnome-compatibility.md` at the repository root is the summary.
