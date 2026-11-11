# Workspaces — how Helix applies the settings

Behavior for `org.gnome.mutter` `dynamic-workspaces` and
`workspaces-only-on-primary`, `org.gnome.desktop.wm.preferences`
`num-workspaces`, the workspace keys of `org.gnome.desktop.wm.keybindings`,
and `current-workspace-only` of `org.gnome.shell.app-switcher` /
`org.gnome.shell.window-switcher` (key tables in the files named after each
schema). Changes apply at once.

The installed mutter schema defaults both mutter keys to false; GNOME Shell
installs an override that makes them true. Helix reads whatever the
installed schemas and the user's settings say.

## Workspaces

Workspaces form one row, numbered from 1 on the left. One is active; only
its windows are shown (and minimized windows stay minimized on their
workspace).

- With `dynamic-workspaces` false there are `num-workspaces` workspaces.
  When the number goes down, windows on workspaces that no longer exist
  move to the last one.
- With `dynamic-workspaces` true there is always exactly one empty
  workspace, the last one: putting a window on it adds a new empty one
  after it, and a workspace that becomes empty is removed unless it is the
  active one or the last one (the workspaces after it move one to the
  left).
- With `workspaces-only-on-primary` true, windows on other monitors than
  the primary one are shown on every workspace; a window that moves onto
  the primary monitor belongs to the workspace it was assigned last (the
  active one when it opened).

New windows open on the active workspace. Activating a window on another
workspace (from a switcher, the launcher, or an app asking for focus)
switches to that workspace first. After switching, the window focused last
on that workspace gets focus.

## Keys

| Keys | Action |
|------|--------|
| `switch-to-workspace-1` … `-12` | switch to that workspace, if it exists |
| `switch-to-workspace-left` / `-right` | the neighbouring workspace; nothing at either end |
| `switch-to-workspace-last` | the last workspace |
| `move-to-workspace-1` … `-12`, `-left`, `-right`, `-last` | move the focused window there and switch with it |

| `toggle-on-all-workspaces` | show the focused window on every workspace, or only on the active one again |

The workspaces are in a row, so the `-up` and `-down` keys do nothing
(see `gaps.md`). A window on every workspace counts as on the active one
for switchers and for dynamic workspaces; when it stops being on every
workspace it belongs to the active one.

## Switchers

With `current-workspace-only` true, the window switcher
(`switch-windows`, `switch-group`) lists only windows on the active
workspace, and the application switcher (`switch-applications`) only
applications with a window there (its entry then activates that
application's most recent window on the active workspace). The defaults
are true for the window switcher and false for the application switcher.
The launcher always lists every window.

## Indicator

When there is more than one workspace, the panel shows one dot per
workspace next to the Apps button, the active one highlighted.
