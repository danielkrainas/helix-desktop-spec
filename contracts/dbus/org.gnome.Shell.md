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

  Helix: the OSD is a pill floating near the bottom of the monitor
  (centered, 72 logical px above the edge, 300 wide, 52 tall; 72 with a
  label and a level), above everything including full-screen windows,
  and the pointer goes through it. It shows the icon (an icon name or a
  serialized themed GIcon; the first name the icon theme has, in its
  symbolic form), then a rounded level bar with the percentage (with a
  label: the label above the bar), or the label alone (or, without a
  label, words for the shell's own icons, e.g. "Microphone off"). With
  `max_level` above 1, a mark on the bar shows where 100 % is. Unknown
  `connector`: every monitor. It stays 2 seconds after the last
  `ShowOSD`; a new one replaces it at once. On the accessibility bus it
  is a window "On-screen display" holding one live `status` object named
  by what it shows ("Volume 45 %"), so each change is announced.
  *GNOME does not document this; Helix's choice:* the sizes, place and
  duration.
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
- `grabFlags` (*GNOME does not document this; Helix's choice:* the
  interface does not say what they are): the key-binding flags of mutter's documented API
  (`Meta.KeyBindingFlags`, which `Meta.Display.grab_accelerator` takes):
  1 per-window, 2 built-in, 4 reversed, 8 always active (non-maskable),
  16 ignore autorepeat, 32 not grabbed automatically, 64 custom trigger,
  128 trigger on release (notify on release as well as press). 0 for
  none.

`AcceleratorActivated(action, parameters)` is emitted when the shortcut is
pressed, and `AcceleratorDeactivated` when it is released if the grab has
the trigger-on-release flag (128).

### Helix

- Only these callers may grab and release accelerators: the owners of
  `org.gnome.SettingsDaemon.MediaKeys` and `org.gnome.Settings` (as GNOME
  Shell allows), and of `org.freedesktop.impl.portal.desktop.helix` (the
  GlobalShortcuts portal backend). Other callers get
  `org.freedesktop.DBus.Error.AccessDenied`.
- *Helix's choice:* keys that type text are never grabbed (action id 0):
  an accelerator with no modifier other than Shift, unless its key is
  never typed (`XF86…` keys, `F1`…`F35`, `Print`, `Pause`,
  `Scroll_Lock`, `Sys_Req`, `Break`, `Help`, `Cancel`). Otherwise a grab
  would see what is typed into apps and the lock screen.
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
In Helix a caller can release only its own grabs: another caller's
action counts as not grabbed (false).
