# `org.helix.extensions`

Helix's own extension settings (not a GNOME schema). Helix installs it as
`org.helix.extensions.gschema.xml`, and this table is its contract.
Behavior: `contracts/extensions/README.md`, "Enabling, approving and
placing".

Path: `/org/helix/extensions/`

| Key | Type | Default |
|-----|------|---------|
| `enabled` | `as` | `[]` |
| `disabled` | `as` | `[]` |
| `disable-all` | `b` | `false` |
| `approved` | `a{sas}` | `{}` |
| `placed` | `as` | `[]` |
| `command-keys` | `a{ss}` | `{}` |
| `require-sandbox` | `b` | `false` |

- `enabled`: extension ids turned on.
- `disabled`: system extensions with `enabled-by-default` turned off.
- `disable-all`: no extension runs.
- `approved`: by extension id, the permissions the user approved.
- `placed`: `<extension id>/<item id>` of items once added to the
  layout (and of commands whose suggested key was considered).
- `command-keys`: by `<extension id>/<command id>`, the command's
  accelerator (`""`: none).
- `require-sandbox`: only extensions with `sandbox = true` run.
