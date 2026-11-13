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
  Helix reports the GNOME version whose surface it implements (`51.0`), not
  its own version, so clients that parse it see a GNOME release. Extension
  tools also read it. The same object serves
  `org.gnome.Shell.Extensions` (`org.gnome.Shell.Extensions.md`): it
  lists and switches extensions, but Helix runs none.
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
  (`i`, wrapped in `v`). Each call replaces the labels shown; monitors not
  in `params` get none. (Helix shows the number in the top-left corner of
  each monitor, below the panel; labels take no input.)
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
- `grabFlags`: the key-binding flags of mutter's documented API
  (`Meta.KeyBindingFlags`, which `Meta.Display.grab_accelerator` takes):
  1 per-window, 2 built-in, 4 reversed, 8 always active (non-maskable),
  16 ignore autorepeat, 32 not grabbed automatically, 64 custom trigger,
  128 trigger on release (notify on release as well as press). 0 for
  none.

`AcceleratorActivated(action, parameters)` is emitted when the shortcut is
pressed, and `AcceleratorDeactivated` when it is released if the grab has
the trigger-on-release flag (128).

### Helix

- Grabs whose `modeFlags` include the lock screen (4) keep working while
  the session is locked; others do not fire then.
- Of the `grabFlags`, only trigger-on-release (128) changes anything:
  without it no `AcceleratorDeactivated` is sent. Global keys never
  autorepeat in Helix (repeat is the focused client's business), so 16
  holds anyway; the other flags concern mutter's own bindings. `parameters` keys:
`device-id` (`u`), `timestamp` (`u`), `action-mode` (`u`), `device-node`
(`s`).

`UngrabAccelerator(action)` / `UngrabAccelerators(actions)` release grabs
and return whether they succeeded (false if an action was not grabbed; for
the list form, false if any was not).
Grabs belong to the caller and are released when it leaves the bus.
