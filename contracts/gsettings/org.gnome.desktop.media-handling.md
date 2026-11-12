# `org.gnome.desktop.media-handling`

Key surface of the `org.gnome.desktop.media-handling` GSettings schema in
gsettings-desktop-schemas 51.0, described through GSettings introspection
of the compiled schema.

Path: `/org/gnome/desktop/media-handling/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `automount` | `b` | `true` |  |
| `automount-open` | `b` | `true` |  |
| `autorun-never` | `b` | `true` |  |
| `autorun-x-content-ignore` | `as` | `[]` |  |
| `autorun-x-content-open-folder` | `as` | `[]` |  |
| `autorun-x-content-start-app` | `as` | `['x-content/unix-software', 'x-content/ostree-repository']` |  |

The `as` keys hold `x-content/*` content types (the shared-mime-info names
GIO guesses for a mounted medium, e.g. `x-content/image-dcf` for a camera
card). Helix's behavior: [removable-media.md](removable-media.md).
