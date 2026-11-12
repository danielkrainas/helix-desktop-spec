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

- [`org.gnome.desktop.background`](org.gnome.desktop.background.md) — Helix's behavior: [background.md](background.md)
- [`org.gnome.desktop.search-providers`](org.gnome.desktop.search-providers.md) — see `contracts/dbus/org.gnome.Shell.SearchProvider2.md`
- [`org.gnome.desktop.app-folders`](org.gnome.desktop.app-folders.md) — see `contracts/wayland/README.md`, "Favorites and app folders"
- [`org.gnome.shell`](org.gnome.shell.md) — `favorite-apps`: see the same section
- [`org.gnome.settings-daemon.peripherals.touchscreen`](org.gnome.settings-daemon.peripherals.touchscreen.md) — see `org.gnome.Mutter.DisplayConfig.md`, "Automatic rotation"
- [`org.gnome.settings-daemon.plugins.power`](org.gnome.settings-daemon.plugins.power.md) — Helix's behavior: [power.md](power.md)
- [`org.gnome.desktop.interface`](org.gnome.desktop.interface.md)
- [`org.gnome.desktop.a11y`](org.gnome.desktop.a11y.md)
- [`org.gnome.desktop.a11y.applications`](org.gnome.desktop.a11y.applications.md) — the on-screen keyboard: [screen-keyboard.md](screen-keyboard.md)
- [`org.gnome.desktop.a11y.interface`](org.gnome.desktop.a11y.interface.md) — Helix's behavior: [a11y-interface.md](a11y-interface.md)
- [`org.gnome.desktop.a11y.keyboard`](org.gnome.desktop.a11y.keyboard.md) — Helix's behavior: [a11y-keyboard.md](a11y-keyboard.md)
- [`org.gnome.desktop.a11y.magnifier`](org.gnome.desktop.a11y.magnifier.md) — Helix's magnifier and screen reader: [a11y-magnifier.md](a11y-magnifier.md)
- [`org.gnome.desktop.a11y.mouse`](org.gnome.desktop.a11y.mouse.md) — Helix's behavior: [a11y-mouse.md](a11y-mouse.md)
- [`org.gnome.desktop.wm.preferences`](org.gnome.desktop.wm.preferences.md)
- [`org.gnome.desktop.session`](org.gnome.desktop.session.md)
- [`org.gnome.desktop.screensaver`](org.gnome.desktop.screensaver.md)
- [`org.gnome.desktop.notifications`](org.gnome.desktop.notifications.md), [`org.gnome.desktop.notifications.application`](org.gnome.desktop.notifications.application.md)
- [`org.gnome.desktop.sound`](org.gnome.desktop.sound.md)
- [`org.gnome.desktop.privacy`](org.gnome.desktop.privacy.md)
- [`org.gnome.desktop.lockdown`](org.gnome.desktop.lockdown.md), [`org.gnome.system.location`](org.gnome.system.location.md)
- [`org.gnome.desktop.break-reminders`](org.gnome.desktop.break-reminders.md), [`org.gnome.desktop.screen-time-limits`](org.gnome.desktop.screen-time-limits.md) — Helix's behavior: [wellbeing.md](wellbeing.md)
- [`org.gnome.desktop.media-handling`](org.gnome.desktop.media-handling.md) — Helix's behavior: [removable-media.md](removable-media.md)
- [`org.gnome.desktop.peripherals.trackball`](org.gnome.desktop.peripherals.trackball.md), [`org.gnome.desktop.peripherals.pointingstick`](org.gnome.desktop.peripherals.pointingstick.md)

From other GNOME 51.0 packages, described the same way:

- [`org.gnome.shell.keybindings`](org.gnome.shell.keybindings.md),
  [`org.gnome.shell.app-switcher`](org.gnome.shell.app-switcher.md),
  [`org.gnome.shell.window-switcher`](org.gnome.shell.window-switcher.md)
  (gnome-shell)
- [`org.gnome.settings-daemon.plugins.media-keys`](org.gnome.settings-daemon.plugins.media-keys.md)
  (gnome-settings-daemon)
- [`org.gnome.mutter`](org.gnome.mutter.md) (mutter)
- [`org.gnome.mutter.keybindings`](org.gnome.mutter.keybindings.md) (mutter)
- [`org.gnome.mutter.wayland`](org.gnome.mutter.wayland.md) (mutter)
- [`org.gnome.mutter.wayland.keybindings`](org.gnome.mutter.wayland.keybindings.md) (mutter)
- [`org.gnome.settings-daemon.plugins.color`](org.gnome.settings-daemon.plugins.color.md)
  (gnome-settings-daemon)

Helix reuses the system dconf and the installed GNOME schemas, so these
tables are what the installed schemas must provide; they are not a
replacement for them. More schemas get tables as Helix starts depending on
them.

## Key tables and behavior notes

Key tables (generated from the compiled 51.0 schemas) exist for the schemas
Helix reads; `input-settings.md` describes how Helix applies the input
schemas, `shell-keys.md` the screenshot, brightness, volume and playback
keys, `workspaces.md` workspaces, and `night-light.md` Night Light.
