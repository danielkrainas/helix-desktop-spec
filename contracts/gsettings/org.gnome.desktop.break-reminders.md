# `org.gnome.desktop.break-reminders`

Key surface of the `org.gnome.desktop.break-reminders` GSettings schemas
in gsettings-desktop-schemas 51.0, described through GSettings
introspection of the compiled schemas. Helix's behavior:
[wellbeing.md](wellbeing.md).

Path: `/org/gnome/desktop/break-reminders/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `selected-breaks` | `as` | `[]` | (break kinds: `eyesight`, `movement`) |

One schema per kind of break, with the same keys:
`org.gnome.desktop.break-reminders.eyesight`
(`/org/gnome/desktop/break-reminders/eyesight/`) and
`org.gnome.desktop.break-reminders.movement`
(`/org/gnome/desktop/break-reminders/movement/`).

| Key | Type | Default (eyesight) | Default (movement) | Allowed values |
|-----|------|--------------------|--------------------|----------------|
| `interval-seconds` | `u` | `1200` | `1800` | 10 and up |
| `duration-seconds` | `u` | `20` | `300` | 10 and up |
| `delay-seconds` | `u` | `180` | `180` | 10 and up |
| `notify` | `b` | `true` | `true` |  |
| `notify-upcoming` | `b` | `false` | `true` |  |
| `notify-overdue` | `b` | `true` | `true` |  |
| `countdown` | `b` | `false` | `false` |  |
| `fade-screen` | `b` | `true` | `true` |  |
| `lock-screen` | `b` | `false` | `false` |  |
| `play-sound` | `b` | `true` | `true` |  |
