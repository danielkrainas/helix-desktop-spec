# GSettings schemas (GNOME 51)

Index of the primary desktop schemas from `gsettings-desktop-schemas` 51.0.
These form the expected settings surface for a compatible session.

Source: https://gitlab.gnome.org/GNOME/gsettings-desktop-schemas (tag 51.0)

## Core schemas (org.gnome.desktop.*)

- `org.gnome.desktop.a11y`
- `org.gnome.desktop.a11y.applications`
- `org.gnome.desktop.a11y.interface`
- `org.gnome.desktop.a11y.keyboard`
- `org.gnome.desktop.a11y.magnifier`
- `org.gnome.desktop.a11y.mouse`
- `org.gnome.desktop.app-folders`
- `org.gnome.desktop.background`
- `org.gnome.desktop.background.login`
- `org.gnome.desktop.break-reminders`
- `org.gnome.desktop.calendar`
- `org.gnome.desktop.datetime`
- `org.gnome.desktop.default-applications`
- `org.gnome.desktop.enums`
- `org.gnome.desktop.input-sources`
- `org.gnome.desktop.interface` (animations, fonts, scaling, etc.)
- `org.gnome.desktop.lockdown`
- `org.gnome.desktop.media-handling`
- `org.gnome.desktop.notifications`
- `org.gnome.desktop.peripherals` (+ children: mouse, touchpad, keyboard, etc.)
- `org.gnome.desktop.privacy`
- `org.gnome.desktop.screen-time-limits`
- `org.gnome.desktop.screensaver`
- `org.gnome.desktop.search-providers`
- `org.gnome.desktop.session`
- `org.gnome.desktop.sound`
- `org.gnome.desktop.thumbnail-cache`
- `org.gnome.desktop.thumbnailers`
- `org.gnome.desktop.wm.keybindings`

Additional shell-specific schemas live in `gnome-shell` (e.g. `org.gnome.shell`).

## Key contracts

Schemas Helix depends on directly have a key table (key, GVariant type,
default, allowed values) for gsettings-desktop-schemas 51.0, described
through GSettings' introspection API from the compiled upstream schemas
rather than copied from schema sources:

- [`org.gnome.desktop.interface`](org.gnome.desktop.interface.md)
- [`org.gnome.desktop.a11y.interface`](org.gnome.desktop.a11y.interface.md)
- [`org.gnome.desktop.wm.preferences`](org.gnome.desktop.wm.preferences.md)
- [`org.gnome.desktop.session`](org.gnome.desktop.session.md)
- [`org.gnome.desktop.screensaver`](org.gnome.desktop.screensaver.md)
- [`org.gnome.desktop.notifications`](org.gnome.desktop.notifications.md)

Helix reuses the system dconf and the installed GNOME schemas, so these
tables are what the installed schemas must provide; they are not a
replacement for them. More schemas get tables as Helix starts depending on
them.
