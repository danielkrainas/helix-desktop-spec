# org.gnome.Shell — behavioral notes

Companion to `org.gnome.Shell.xml`.

## Bus name and object

- Bus name: `org.gnome.Shell`
- Object: `/org/gnome/Shell`

## Properties

- `Mode` (`s`): the session mode. `user` for a normal user session (other
  modes such as `gdm` and `unlock-dialog` belong to the login screen and lock
  screen).
- `ShellVersion` (`s`): a GNOME-style version string such as `51.0`.
- `OverviewActive` (`b`, read/write): whether the overview is shown. Setting
  it shows or hides the overview.

## Methods

- `Eval(script)`: returns `(false, "")` unless the shell runs in an
  explicitly unsafe/development mode. Clients must not rely on it.
- `FocusSearch`: shows the overview with the search entry focused.
- `ShowApplications`: shows the overview's application grid.
- `FocusApp(id)`: shows the application grid with the app `id` (a desktop
  file id) selected.
- `ShowOSD(params)`: shows an on-screen display. `params` keys, all optional:
  - `icon` (`s`): icon name or serialized GIcon
  - `label` (`s`)
  - `level` (`d`): current level, 0 to `max_level`
  - `max_level` (`d`): defaults to 1
  - `connector` (`s`): monitor connector to show it on (all if absent)
- `ShowMonitorLabels(params)` / `HideMonitorLabels`: show/hide labels
  identifying monitors; `params` maps connector names (`s`) to label numbers
  (`i`, wrapped in `v`).
- `ScreenTransition`: briefly freezes the screen so a following change
  (e.g. a display reconfiguration) appears as a cross-fade.

## Accelerators

`GrabAccelerator(accelerator, modeFlags, grabFlags)` registers a global
keyboard shortcut and returns an action id; `0` means the grab failed (e.g.
the accelerator is invalid or already taken). `GrabAccelerators` does the
same for a list, returning ids in order.

- `accelerator` uses GTK accelerator syntax, e.g. `<Super>a`,
  `<Control><Alt>Delete`, `XF86AudioMute`.
- `modeFlags` (action modes in which the shortcut works): 1 normal, 2
  overview, 4 lock screen, 8 unlock screen, 16 login screen, 32 system
  modal dialog, 64 looking glass, 128 top-bar popup; `0xFFFFFFFF`
  (all).
- `grabFlags`: 0 for none.

`AcceleratorActivated(action, parameters)` is emitted when the shortcut is
pressed and `AcceleratorDeactivated` when released. `parameters` keys:
`device-id` (`u`), `timestamp` (`u`), `action-mode` (`u`), `device-node`
(`s`).

`UngrabAccelerator(action)` / `UngrabAccelerators(actions)` release grabs.
Grabs belong to the caller and are released when it leaves the bus.
