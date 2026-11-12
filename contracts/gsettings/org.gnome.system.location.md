# `org.gnome.system.location`

Key surface of the `org.gnome.system.location` GSettings schema in gsettings-desktop-schemas
51.0, described through GSettings introspection of the compiled schema.

Path: `/org/gnome/system/location/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `enabled` | `b` | `false` |  |
| `max-accuracy-level` | `s` | `'exact'` | `country`, `city`, `neighborhood`, `street`, `exact` |

Helix's use: `contracts/dbus/org.freedesktop.GeoClue2.Agent.md`; the
Lockdown portal reports `disable-location` as the opposite of `enabled`.
