# `org.gnome.shell.extensions.dash-to-dock`

Key surface of the `org.gnome.shell.extensions.dash-to-dock` GSettings
schema of Ubuntu's dock (Dash to Dock), as Ubuntu 24.04 installs it
(package `gnome-shell-extension-ubuntu-dock` 90ubuntu3), described through
GSettings introspection of the compiled schema. It is not a GNOME schema:
Helix's dock follows it when it is installed (`contracts/wayland/README.md`,
"Dock" lists the keys Helix uses).

Path: `/org/gnome/shell/extensions/dash-to-dock/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `activate-single-window` | `b` | `true` |  |
| `always-center-icons` | `b` | `false` |  |
| `animate-show-apps` | `b` | `true` |  |
| `animation-time` | `d` | `0.20000000000000001` |  |
| `app-ctrl-hotkey-1` | `as` | `['<Ctrl><Super>1']` |  |
| `app-ctrl-hotkey-10` | `as` | `['<Ctrl><Super>0']` |  |
| `app-ctrl-hotkey-2` | `as` | `['<Ctrl><Super>2']` |  |
| `app-ctrl-hotkey-3` | `as` | `['<Ctrl><Super>3']` |  |
| `app-ctrl-hotkey-4` | `as` | `['<Ctrl><Super>4']` |  |
| `app-ctrl-hotkey-5` | `as` | `['<Ctrl><Super>5']` |  |
| `app-ctrl-hotkey-6` | `as` | `['<Ctrl><Super>6']` |  |
| `app-ctrl-hotkey-7` | `as` | `['<Ctrl><Super>7']` |  |
| `app-ctrl-hotkey-8` | `as` | `['<Ctrl><Super>8']` |  |
| `app-ctrl-hotkey-9` | `as` | `['<Ctrl><Super>9']` |  |
| `app-hotkey-1` | `as` | `['<Super>1']` |  |
| `app-hotkey-10` | `as` | `['<Super>0']` |  |
| `app-hotkey-2` | `as` | `['<Super>2']` |  |
| `app-hotkey-3` | `as` | `['<Super>3']` |  |
| `app-hotkey-4` | `as` | `['<Super>4']` |  |
| `app-hotkey-5` | `as` | `['<Super>5']` |  |
| `app-hotkey-6` | `as` | `['<Super>6']` |  |
| `app-hotkey-7` | `as` | `['<Super>7']` |  |
| `app-hotkey-8` | `as` | `['<Super>8']` |  |
| `app-hotkey-9` | `as` | `['<Super>9']` |  |
| `app-shift-hotkey-1` | `as` | `['<Shift><Super>1']` |  |
| `app-shift-hotkey-10` | `as` | `['<Shift><Super>0']` |  |
| `app-shift-hotkey-2` | `as` | `['<Shift><Super>2']` |  |
| `app-shift-hotkey-3` | `as` | `['<Shift><Super>3']` |  |
| `app-shift-hotkey-4` | `as` | `['<Shift><Super>4']` |  |
| `app-shift-hotkey-5` | `as` | `['<Shift><Super>5']` |  |
| `app-shift-hotkey-6` | `as` | `['<Shift><Super>6']` |  |
| `app-shift-hotkey-7` | `as` | `['<Shift><Super>7']` |  |
| `app-shift-hotkey-8` | `as` | `['<Shift><Super>8']` |  |
| `app-shift-hotkey-9` | `as` | `['<Shift><Super>9']` |  |
| `application-counter-overrides-notifications` | `b` | `true` |  |
| `apply-custom-theme` | `b` | `false` |  |
| `apply-glossy-effect` | `b` | `true` |  |
| `autohide` | `b` | `true` |  |
| `autohide-in-fullscreen` | `b` | `false` |  |
| `background-color` | `s` | `'#ffffff'` |  |
| `background-opacity` | `d` | `0.80000000000000004` |  |
| `bolt-support` | `b` | `true` |  |
| `click-action` | `s` | `'cycle-windows'` | `skip`, `minimize`, `launch`, `cycle-windows`, `minimize-or-overview`, `previews`, `minimize-or-previews`, `focus-or-previews`, `focus-or-appspread`, `focus-minimize-or-previews`, `focus-minimize-or-appspread`, `quit` |
| `custom-background-color` | `b` | `false` |  |
| `custom-theme-customize-running-dots` | `b` | `false` |  |
| `custom-theme-running-dots-border-color` | `s` | `'#ffffff'` |  |
| `custom-theme-running-dots-border-width` | `i` | `0` |  |
| `custom-theme-running-dots-color` | `s` | `'#ffffff'` |  |
| `custom-theme-shrink` | `b` | `false` |  |
| `customize-alphas` | `b` | `false` |  |
| `dance-urgent-applications` | `b` | `true` |  |
| `dash-max-icon-size` | `i` | `48` |  |
| `default-windows-preview-to-open` | `b` | `false` |  |
| `disable-overview-on-startup` | `b` | `false` |  |
| `dock-fixed` | `b` | `false` |  |
| `dock-position` | `s` | `'BOTTOM'` | `TOP`, `RIGHT`, `BOTTOM`, `LEFT` |
| `extend-height` | `b` | `false` |  |
| `force-straight-corner` | `b` | `false` |  |
| `height-fraction` | `d` | `0.90000000000000002` |  |
| `hide-delay` | `d` | `0.20000000000000001` |  |
| `hide-tooltip` | `b` | `false` |  |
| `hot-keys` | `b` | `true` |  |
| `hotkeys-overlay` | `b` | `true` |  |
| `hotkeys-show-dock` | `b` | `true` |  |
| `icon-size-fixed` | `b` | `false` |  |
| `intellihide` | `b` | `true` |  |
| `intellihide-mode` | `s` | `'FOCUS_APPLICATION_WINDOWS'` | `ALL_WINDOWS`, `FOCUS_APPLICATION_WINDOWS`, `MAXIMIZED_WINDOWS`, `ALWAYS_ON_TOP` |
| `isolate-locations` | `b` | `true` |  |
| `isolate-monitors` | `b` | `false` |  |
| `isolate-workspaces` | `b` | `false` |  |
| `manualhide` | `b` | `false` |  |
| `max-alpha` | `d` | `0.80000000000000004` |  |
| `middle-click-action` | `s` | `'launch'` | `skip`, `minimize`, `launch`, `cycle-windows`, `minimize-or-overview`, `previews`, `minimize-or-previews`, `focus-or-previews`, `focus-or-appspread`, `focus-minimize-or-previews`, `focus-minimize-or-appspread`, `quit` |
| `min-alpha` | `d` | `0.20000000000000001` |  |
| `minimize-shift` | `b` | `true` |  |
| `multi-monitor` | `b` | `false` |  |
| `preferred-monitor` | `i` | `-2` |  |
| `preferred-monitor-by-connector` | `s` | `'primary'` |  |
| `pressure-threshold` | `d` | `100.0` |  |
| `preview-size-scale` | `d` | `0.0` |  |
| `require-pressure-to-show` | `b` | `true` |  |
| `running-indicator-dominant-color` | `b` | `false` |  |
| `running-indicator-style` | `s` | `'DEFAULT'` | `DEFAULT`, `DOTS`, `SQUARES`, `DASHES`, `SEGMENTED`, `SOLID`, `CILIORA`, `METRO`, `BINARY` |
| `scroll-action` | `s` | `'do-nothing'` | `do-nothing`, `cycle-windows`, `switch-workspace` |
| `scroll-switch-workspace` | `b` | `true` |  |
| `scroll-to-focused-application` | `b` | `true` |  |
| `shift-click-action` | `s` | `'minimize'` | `skip`, `minimize`, `launch`, `cycle-windows`, `minimize-or-overview`, `previews`, `minimize-or-previews`, `focus-or-previews`, `focus-or-appspread`, `focus-minimize-or-previews`, `focus-minimize-or-appspread`, `quit` |
| `shift-middle-click-action` | `s` | `'launch'` | `skip`, `minimize`, `launch`, `cycle-windows`, `minimize-or-overview`, `previews`, `minimize-or-previews`, `focus-or-previews`, `focus-or-appspread`, `focus-minimize-or-previews`, `focus-minimize-or-appspread`, `quit` |
| `shortcut` | `as` | `['<Super>q']` |  |
| `shortcut-text` | `s` | `'<Super>q'` |  |
| `shortcut-timeout` | `d` | `2.0` |  |
| `show-apps-always-in-the-edge` | `b` | `true` |  |
| `show-apps-at-top` | `b` | `false` |  |
| `show-delay` | `d` | `0.25` |  |
| `show-dock-urgent-notify` | `b` | `true` |  |
| `show-favorites` | `b` | `true` |  |
| `show-icons-emblems` | `b` | `true` |  |
| `show-icons-notifications-counter` | `b` | `true` |  |
| `show-mounts` | `b` | `true` |  |
| `show-mounts-network` | `b` | `false` |  |
| `show-mounts-only-mounted` | `b` | `true` |  |
| `show-running` | `b` | `true` |  |
| `show-show-apps-button` | `b` | `true` |  |
| `show-trash` | `b` | `true` |  |
| `show-windows-preview` | `b` | `true` |  |
| `transparency-mode` | `s` | `'DEFAULT'` | `DEFAULT`, `FIXED`, `DYNAMIC` |
| `unity-backlit-items` | `b` | `false` |  |
| `workspace-agnostic-urgent-windows` | `b` | `true` |  |
