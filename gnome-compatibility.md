# GNOME compatibility

What a Helix session offers to software and users that expect GNOME 51:
what it supports, what it does not (and why, where there is a reason),
and where it behaves differently on purpose. It summarises the contracts
in `contracts/`, which stay the source of truth for details; open work is
tracked as issues in helix-desktop (labels `gap` and `needs-hardware`).
`SCOPE.md` sets the boundaries this follows.

In short: Helix aims to run unmodified GNOME applications and tools and
to honour the user's GNOME settings, through GNOME's public D-Bus
interfaces, GSettings schemas, portals and Wayland protocols. It does not
copy GNOME Shell's look, its internal APIs or its JavaScript extensions.

## Supported

### Applications

- GTK 3, GTK 4 and libadwaita applications run unmodified, Wayland or X11
  (through Xwayland, with Helix's own window frames and scaling).
  Chromium/Electron and Qt applications run too.
- The Wayland protocols those toolkits use, plus GNOME 51's expectations
  (`contracts/wayland/README.md`): layer shell, primary selection,
  fractional scaling, colour management and HDR, tablets, gestures, input
  methods, keyboard-shortcut inhibition, activation tokens, toplevel icons
  and more.
- xdg-desktop-portal backends: Settings, Inhibit, Screenshot, ScreenCast,
  RemoteDesktop, Clipboard, Access, Account, AppChooser, Background,
  Wallpaper, Lockdown, Notification (v2), Print, FileChooser, Email,
  DynamicLauncher, InputCapture (v2) and GlobalShortcuts (v1). Other
  interfaces go to the GTK backend (`contracts/dbus/org.freedesktop.impl.portal.md`).
  The Settings portal carries the colour scheme, accent colour, contrast
  and `org.gnome.desktop.interface` keys that sandboxed GTK reads.

### Session and shell D-Bus interfaces

GNOME's own interfaces, so that GNOME Settings, gnome-control-center
panels, GNOME apps and command-line tools work against Helix:

- Session: `org.gnome.SessionManager` (clients, inhibitors, logout,
  restart and power off through logind, restarting into the firmware
  setup, autostart, session restore) and
  its end-session dialog.
- Shell: `org.gnome.Shell` (accelerator grabs, OSD, overview state;
  `Eval` always refused), `org.gnome.Shell.Screenshot`,
  `org.gnome.Shell.Screencast`, `org.gnome.Shell.Extensions` (listing and
  the extension keys only, see below), search providers
  (`org.gnome.Shell.SearchProvider2`), `org.gnome.ScreenSaver`.
- Mutter: `DisplayConfig` (monitor layout, scaling, rotation, colour
  modes, colour profiles, night light gamma, backlight including DDC/CI,
  underscanning, privacy screens), `IdleMonitor`, `ScreenCast` and
  `RemoteDesktop` (with clipboard and keymaps).
- `org.gnome.SettingsDaemon.Color` (night light state).
- Agents and prompts the shell provides: polkit authentication,
  gnome-keyring/gcr prompts, NetworkManager secrets (Wi-Fi, 802.1X,
  mobile broadband, VPN plugins, WireGuard), Bluetooth pairing and OBEX
  file transfer, GeoClue location, GTK mount operations.
- Desktop-wide services apps talk to: notifications
  (`org.freedesktop.Notifications`, `org.gtk.Notifications`), MPRIS media
  controls, status icons (StatusNotifierItem), Unity launcher entries
  (badges and progress in the dock), IBus input methods, calendar events
  from Evolution Data Server.
- Accessibility: the AT-SPI bus is started as GNOME does, the shell
  publishes its UI on it, and the compositor serves
  `org.freedesktop.a11y.KeyboardMonitor` for screen readers.

### Settings

Helix reads and writes the user's GNOME settings through the system's
GSettings schemas, so switching between GNOME and Helix keeps them. The
schemas covered are listed in `contracts/gsettings/`, including:

- interface, background, screensaver, lock-down, privacy, session,
  sound, notifications and per-app notification settings, app folders;
- all peripherals (keyboard, mouse, touchpad, trackball, pointing stick,
  touchscreen, tablets, styluses and pads), input sources;
- window-management preferences and key bindings (`org.gnome.desktop.wm.*`,
  `org.gnome.mutter.*`, `org.gnome.shell.keybindings`), media keys;
- accessibility (keyboard, mouse, magnifier, interface, applications);
- power, colour and night light, location, search providers,
  break reminders and screen-time limits, removable media handling;
- `org.gnome.shell` (favourites, app picker layout), the app and window
  switchers, and Ubuntu's Dash to Dock settings for the dock.

### Desktop features

The shell provides the features GNOME users expect, each in its own
design (see "Differs on purpose"): top panel, quick settings, the
notification list with calendar, world clocks, weather and media
controls, overview and dynamic workspaces, the application grid with
folders, a dock, application and window switchers, window tiling,
screenshots and screen recording, the lock screen, an on-screen keyboard,
magnifier, high contrast, Large Text, reduced motion, sticky/slow/bounce
and mouse keys, dwell click, hot corner, Wellbeing break reminders,
desktop icons, removable-media handling, and privacy indicators.

## Not supported

| What | Why |
|---|---|
| GNOME Shell extensions (JavaScript) | They patch GNOME Shell's internal, unstable APIs; supporting them means continually reverse-engineering private code. `org.gnome.Shell.Extensions` lists them and keeps their keys so tools don't break, but none runs. Helix has its own isolated extension system instead (`proposals/0001-extensions.md`, `contracts/extensions/`). |
| Exact visual, animation and theme parity with GNOME Shell | Out of scope by design (`SCOPE.md`); GNOME Shell's look is not a public contract. Applications keep GNOME's look because GTK and libadwaita draw them. |
| GDM | GDM's login screen *is* GNOME Shell in a login mode, and distributions make GDM depend on it. Helix can still be chosen from GDM, and ships its own greetd greeter (`components/greeter.md`). |
| X11 sessions from the Helix greeter | greetd starts no X server; X11 sessions would need a wrapper that starts one (helix-desktop#76). |
| Accessibility on the Helix login screen | Not written yet (screen reader, on-screen keyboard, magnifier, high contrast; helix-desktop#77). |
| Translations of the shell | Not written yet: the shell's text, clock and calendar names are English (helix-desktop#81). |
| Some shell UI on the accessibility bus | Not written yet: the overview, pop-up menus, workspace popup, on-screen keyboard, screenshot UI and a few dialogs are not published to screen readers (`contracts/dbus/org.a11y.atspi.md`, "Not covered"; helix-desktop#73). |
| `org.freedesktop.a11y.PointerLocator` | Not served yet (helix-desktop#75). |
| Mutter's `ChangeBacklight` | Deprecated in GNOME; Helix serves the current `Backlight` API only. |
| Mutter's input capture interface (`org.gnome.Mutter.InputCapture`) | Undocumented; the InputCapture portal is served through Helix's own private interface instead. |
| `ApplyMonitorsConfig` properties other than `color-mode` and `underscanning` (e.g. RGB range) | Their key names are not documented. |
| Colour curves `st240`, `xvycc`, `st428` | Not written; other named curves, ICC profiles and Windows-scRGB are converted. HDR output is 8 bits per channel (Smithay picks one scanout format per GPU). Both are helix-desktop#80. |
| Vertical workspace keys (`-up`/`-down`) | Workspaces are laid out in a row, as in GNOME 51, so they have nothing to do. |
| `disable-workarounds` | Has no effect; Helix has no workarounds to disable. |
| Compositing-manager selection for X11 apps | X11 GTK windows with client-side decorations can show dark corners: Helix does not claim `_NET_WM_CM_S<n>` or read `_GTK_FRAME_EXTENTS` (helix-desktop#72). Wayland windows are unaffected. |
| Bluetooth device settings in the shell | Pairing and connecting only; device settings belong to Settings. |
| Desktop Icons NG's own settings | That is an Ubuntu extension; Helix shows a simple set of desktop icons from `show-desktop-icons` only. |
| GNOME's private helpers and files | Not public contracts: `gnome-shell-calendar-server` (Helix reads Evolution Data Server directly), libgweather (Helix asks MET Norway directly), the Wellbeing history file, IBus emoji and Unicode modes in the shell (left to IBus itself). |
| Login screen details GDM has | AccountsService user pictures and per-user session memory, the `clock-format` setting, the primary monitor and localised session names are not used by the greeter (helix-desktop#79). |
| Greeter on Fedora and Arch | Packaged for Debian and Ubuntu only so far; greetd's user and paths differ elsewhere (helix-desktop#78). |

## Differs on purpose

Much of GNOME Shell's and Mutter's behaviour is not documented anywhere
public. Where a contract leaves something open, Helix makes its own
choice and writes it into the contract, rather than copying GNOME's
code. The choices that users or developers are most likely to notice:

### Look and feel

- **Shell design.** The panel, quick settings, notification list,
  overview, app grid, switchers, dialogs, lock screen, screenshot UI,
  on-screen keyboard and animations are Helix's own designs, built on
  shared colour tokens and accessibility rules (WCAG AA text contrast,
  visible focus rings, reduced motion). GNOME's are not public contracts.
- **A dock.** Vanilla GNOME has none; Helix's follows a subset of
  Ubuntu's Dash to Dock settings. It shows window titles in tooltips
  instead of hover previews, previews on click, and quicklists inside the
  dock menu.
- **Window frames for X11 apps**, the window menu and its keyboard
  Move/Resize keys (arrows move 10 pixels, 1 with Control), and window
  shading are Helix's own.
- **Lock screen.** Its own resting state, layout, cards and transitions;
  GNOME's shield and unlock dialog are not documented.

### Safety and robustness

- **Dialogs default to the safe button.** Enter means Wait for an
  unresponsive app, Deny for remote control and shortcut requests, Cancel
  for extension approval; keys pressed in the first 300 ms after a dialog
  appears are ignored.
- **Session manager.** A crashed app's clients and inhibitors are dropped
  when its connection goes away, so it cannot block logout forever; the
  session ends cleanly if the bus or session manager goes away.
- **Display changes ask through Helix's own confirmation dialog**
  (`org.helix.DisplayChangeDialog`); GNOME's happens inside gnome-shell.
- **Keyboard monitor stands aside while locked**, so whoever owns the
  screen reader's name cannot read the password; VT-switching keys can
  never be grabbed.
- **Screenshot callers are restricted** to the portal backend and the
  GNOME Screenshot app; non-interactive portal screenshots ask the user
  unless the frontend says the app's permission was checked.
- **Portals avoid GNOME's backend.** Interfaces Helix does not serve go
  to the GTK backend, because GNOME's backend needs GNOME Shell and
  stalled apps for 25 seconds when it could not start.

### Behaviour where GNOME is undocumented

- **Session startup** uses Helix's own launcher and a systemd user target;
  components announce readiness by owning a bus name or calling
  `SessionManager.Setenv`.
- **Idle**: every input event counts as user activity.
- **Window switching**: application and window ordering, time-based
  `cycle-windows`/`cycle-group`, and `switch-panels` putting keyboard
  focus on the panel (GNOME shows a popup).
- **Workspaces and overview**: dynamic workspace rules, overview
  pictures renewed twice a second, the workspace switch popup.
- **Hot corner**: 100 unaccelerated pixels of push within a second.
- **Scaling**: fractional scales in 0.25 steps that give whole logical
  sizes; captures spanning monitors use the highest scale; Xwayland is
  scaled by the compositor, not Xwayland's own HiDPI mode.
- **Brightness and volume keys**: brightness steps of 1/20 of the range on
  every controllable monitor, precise volume steps of 1 %, OSD icons
  switching at 33 % and 66 %.
- **Night light, power, magnifier, dwell click, sticky keys and other
  accessibility features**: Helix's own curves, timings and formulas
  where GNOME's are not documented.
- **Notifications**: Helix's own message list, grouping and stacking,
  and its own paths for per-application settings.
- **Polkit**: when several identities can authenticate, Helix asks which
  first, in a dialog of its own.
- **Location**: Helix registers with GeoClue as `gnome-shell`, because
  GeoClue only accepts agents its configuration lists.
- **Weather and world clocks**: MET Norway forecasts; an automatic
  location and a world clock's time zone come from the nearest
  `zone1970.tab` city (which can be wrong near zone borders).
- **Calendar recurrences** are expanded by Helix itself, from the rule
  parts listed in `contracts/gsettings/calendar.md`.
- **On-screen keyboard**: letters and symbols from the active layout,
  one accent table for all languages, hunspell suggestions, and typed
  words kept in memory only.
- **Status icons** use the `org.kde.*` names apps actually use, although
  the specification says `org.freedesktop.*`.
- **Login screen**: the greeter shows initials instead of pictures, a
  24-hour clock, and remembers the last user and session itself.

Each of these is written down in its contract, marked *GNOME does not
document this; Helix's choice:* where GNOME is silent, so other
implementations can follow or differ knowingly.

## Not yet verified

Some supported features are written to their published standards and
tested against simulated devices or services, but not yet with the real
thing. Each contract says so where it applies, and the full list is the
helix-desktop issues labelled `needs-hardware`; the main ones are touchscreens and touchpad gestures,
drawing tablets, HDR and calibrated monitors, DDC/CI brightness, privacy
screens, underscanning, offline updates, user switching with a real
display manager, Bluetooth file transfer, USB protection, and the
greeter's start through greetd. Screen-reader support is tested over
AT-SPI on a private bus but not yet with Orca on a Helix session.
