# `org.gnome.desktop.media-handling`

Key surface of the `org.gnome.desktop.media-handling` GSettings schema,
described through GSettings introspection of a compiled schema. The
compiled schema available was gsettings-desktop-schemas 46 with a
distribution's defaults, not 51.0; see `gaps.md`.

Path: `/org/gnome/desktop/media-handling/`

| Key | Type | Default (as found) | Allowed values |
|-----|------|--------------------|----------------|
| `automount` | `b` | `true` |  |
| `automount-open` | `b` | `true` |  |
| `autorun-never` | `b` | `true` |  |
| `autorun-x-content-ignore` | `as` | `[]` |  |
| `autorun-x-content-open-folder` | `as` | `[]` |  |
| `autorun-x-content-start-app` | `as` | `['x-content/unix-software', 'x-content/ostree-repository']` |  |

The `as` keys hold `x-content/*` content types (the shared-mime-info names
GIO guesses for a mounted medium, e.g. `x-content/image-dcf` for a camera
card). Helix's behavior: [removable-media.md](removable-media.md).
