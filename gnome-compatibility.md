# GNOME compatibility

What a Helix session offers to software and users that expect GNOME 51:
what it supports, what it does not (and why, where there is a reason),
and where it behaves differently on purpose. It summarises the contracts
in `contracts/`, which stay the source of truth for details; open work is
tracked as issues in helixdesktop/desktop (labels `gap` and `needs-hardware`).
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
  publishes all of its UI on it (panel, menus, overview, launcher,
  dialogs, notifications, dock, on-screen keyboard, screenshot UI; text
  fields with character extents and editing; `contracts/dbus/org.a11y.atspi.md`),
  and the compositor serves
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

The system sounds volume, which GNOME keeps in the sound server rather
than in GSettings, is shared too: Helix's Settings app uses the same
stream-restore entry under PipeWire and WirePlumber
(`contracts/gsettings/org.gnome.desktop.sound.md`).

### Desktop features

The shell provides the features GNOME users expect, each in its own
design (see "Differs on purpose"): top panel, quick settings, the
notification list with calendar, world clocks, weather and media
controls, overview and dynamic workspaces, the application grid with
folders, a dock, application and window switchers, window tiling,
screenshots and screen recording, the lock screen, an on-screen keyboard,
magnifier, high contrast, Large Text, reduced motion, sticky/slow/bounce
and mouse keys, dwell click, hot corner, Wellbeing break reminders,
desktop icons, the desktop background's menu (Change Background…,
Display Settings, Settings), removable-media handling, and privacy
indicators.

### Languages

The shell, lock screen, the system dialogs the shell draws, the portals'
own dialogs and the login screen are translatable (gettext, one
catalogue, with plural forms) and follow the user's locale, or the
system's for the login screen: month and weekday names, translatable
date and time patterns with `clock-format`, the locale's first weekday,
number, percentage and size formats, and accessible names. Right-to-left
languages mirror the shell's own layouts and draw bidirectional text
(`components/i18n.md`). Helix ships translations for Spanish, French,
German, Brazilian Portuguese, Russian, Simplified Chinese, Japanese,
Italian, Polish, Dutch, Korean and Ukrainian; *differs on purpose*: they
are machine translations shipped before review by native speakers
(GNOME's come from its translation teams), labelled as unreviewed in
each catalogue and in `TRANSLATING.md`. Chinese and Japanese text wraps
between characters; CJK text needs the Noto CJK fonts.

## Not supported

| What | Why |
|---|---|
| GNOME Shell extensions (JavaScript) | They patch GNOME Shell's internal, unstable APIs; supporting them means continually reverse-engineering private code. `org.gnome.Shell.Extensions` lists them and keeps their keys so tools don't break, but none runs. Helix has its own isolated extension system instead (`proposals/0001-extensions.md`, `contracts/extensions/`). |
| Exact visual, animation and theme parity with GNOME Shell | Out of scope by design (`SCOPE.md`); GNOME Shell's look is not a public contract. Applications keep GNOME's look because GTK and libadwaita draw them. |
| GDM | GDM's login screen *is* GNOME Shell in a login mode, and distributions make GDM depend on it. Helix can still be chosen from GDM, and ships its own greetd greeter (`components/greeter.md`). |
| X11 sessions from the Helix greeter | greetd starts no X server; X11 sessions would need a wrapper that starts one (helixdesktop/desktop#76). |
| Magnifier on the Helix login screen | The greeter offers high contrast, large text, the screen reader and its own on-screen keyboard, but no zoom (`components/greeter.md`, "Accessibility"). |
| Text shaping for Arabic and Indic scripts | Helix's own text renderer has no shaping engine: those scripts draw letter by letter (Hebrew and scripts without shaping are fine). Right-to-left surfaces also publish left-to-right bounds on the accessibility bus and keep left-to-right arrow-key order (`components/i18n.md`, "Not done yet"). |
| `org.freedesktop.a11y.PointerLocator` | Served (`contracts/dbus/org.freedesktop.a11y.KeyboardMonitor.md`); not yet tried with Orca's mouse review. |
| Mutter's `ChangeBacklight` | Deprecated in GNOME; Helix serves the current `Backlight` API only. |
| Mutter's input capture interface (`org.gnome.Mutter.InputCapture`) | Undocumented; the InputCapture portal is served through Helix's own private interface instead. |
| `ApplyMonitorsConfig` properties other than `color-mode` and `underscanning` (e.g. RGB range) | Their key names are not documented. |
| Mutter's `monitors.xml` (user's or `/etc/xdg/monitors.xml`) | Helix remembers layouts in its own `helix/monitors.toml`, with the same lookup (the user's, then `$XDG_CONFIG_DIRS`); the login screen's layout is set from Settings or by copying into `/etc/xdg/helix/monitors.toml` (`contracts/dbus/org.gnome.Mutter.DisplayConfig.md`, "Remembered configurations"; `components/greeter.md`, "The monitor"). |
| Colour curves `st240`, `xvycc`, `st428` | Not written; other named curves, ICC profiles and Windows-scRGB are converted. HDR output is 8 bits per channel (Smithay picks one scanout format per GPU). Both are helixdesktop/desktop#80. |
| Vertical workspace keys (`-up`/`-down`) | Workspaces are laid out in a row, as in GNOME 51, so they have nothing to do. |
| `disable-workarounds` | Has no effect; Helix has no workarounds to disable. |
| Client-side shadows of X11 GTK windows | Helix does not advertise `_GTK_FRAME_EXTENTS`, so X11 GTK windows draw no shadow of their own (their transparency, e.g. rounded corners, is blended). Wayland windows are unaffected. |
| Bluetooth device settings in the shell | Pairing and connecting only; device settings belong to Settings. |
| Desktop Icons NG's own settings | That is an Ubuntu extension; Helix shows a simple set of desktop icons from `show-desktop-icons` only. |
| GNOME's private helpers and files | Not public contracts: `gnome-shell-calendar-server` (Helix reads Evolution Data Server directly), libgweather (Helix asks MET Norway directly), the Wellbeing history file, IBus emoji and Unicode modes in the shell (left to IBus itself). |
| Greeter on Fedora and Arch | Packaged for Debian and Ubuntu only so far; greetd's user and paths differ elsewhere (helixdesktop/desktop#78). |

## Differs on purpose

Much of GNOME Shell's and Mutter's behaviour is not documented anywhere
public. Where a contract leaves something open, Helix makes its own
choice and writes it into the contract, rather than copying GNOME's
code. The choices that users or developers are most likely to notice:

### Look and feel

- **Shell design.** The panel, quick settings, notification list,
  overview, app grid, switchers, dialogs, lock screen, screenshot UI,
  on-screen keyboard and animations are Helix's own designs, built on
  shared colour, shape, type and spacing tokens and one set of components
  (rounded popovers with soft shadows, buttons, rows, switches) and on
  accessibility rules (WCAG AA text contrast, visible focus rings, reduced
  motion). They follow GNOME 51's look in spirit; GNOME's are not public
  contracts.
- **A dock.** Vanilla GNOME has none; Helix's follows a subset of
  Ubuntu's Dash to Dock settings. It shows window titles in tooltips
  instead of hover previews, previews on click, and quicklists inside the
  dock menu.
- **Window frames for X11 apps**, the window menu and its keyboard
  Move/Resize keys (arrows move 10 pixels, 1 with Control), and window
  shading are Helix's own.
- **Lock screen.** Its own resting state, layout, cards and transitions;
  GNOME's shield and unlock dialog are not documented.
- **Recent apps in the app grid.** The app grid's first page starts with
  a "Recent" row of the apps launched last (GNOME 51 has none). It
  follows `remember-app-usage`, can be turned off with Helix's own
  `org.helixdesktop.shell app-grid-recent`, and can be cleared from its menu;
  see `contracts/wayland/README.md`, "App grid".

- **Layer shell for every client.** GNOME offers no layer shell; Helix
  offers `zwlr_layer_shell_v1` to all clients, so third-party bars, docks
  and game overlays work. Over a full-screen game, top-layer surfaces are
  hidden and overlay-layer ones (a HUD) stay above it; click-through
  overlays let input reach the game, and exclusive zones never shrink a
  full-screen window (`contracts/wayland/README.md`, "Layer shell for
  other clients"). Tested headless with stand-in clients, not with real
  games or MangoHud.
- **Tearing for full-screen games.** GNOME 51 has no tearing control;
  Helix offers `wp_tearing_control_v1`, and on the DRM backend a focused
  full-screen game that asks (or an X11 game through Xwayland with vsync
  off) is shown without waiting for vblank when nothing else is drawn
  over it and the GPU can; `org.helixdesktop.shell allow-tearing` turns it off
  (`contracts/wayland/README.md`, "Tearing"). Not yet tried on real GPUs
  (helixdesktop/desktop#110).

- **File chooser.** The FileChooser portal's dialog is drawn by the
  shell in Helix's look, modelled on GTK 4's chooser (path bar, places,
  a list with Name, Size and Modified sorted by the headers, type-ahead,
  double-click, a file type menu, Save's name at the top, New Folder),
  and shares the sort order and hidden files with GTK 4 through
  `org.gtk.gtk4.Settings.FileChooser`. Its size comes from the screen
  (it is not resizable), folders are read off the UI thread with a
  "Loading…" state, and it has no grid view, Type column, Starred, Trash
  or Other Locations, or eject buttons yet (helixdesktop/desktop#118;
  `contracts/dbus/org.freedesktop.impl.portal.md`, "FileChooser").

### Safety and robustness

- **Dialogs default to the safe button.** Enter means Wait for an
  unresponsive app, Deny for remote control and shortcut requests, Cancel
  for extension approval; keys pressed in the first 300 ms after a dialog
  appears are ignored.
- **Session manager.** A crashed app's clients and inhibitors are dropped
  when its connection goes away, so it cannot block logout forever; the
  session ends cleanly if the bus or session manager goes away.
- **Display changes ask through Helix's own confirmation dialog**
  (`org.helixdesktop.DisplayChangeDialog`); GNOME's happens inside gnome-shell.
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
- **Login screen**: the greeter reads each user's session from
  AccountsService but remembers sessions itself (it cannot write them
  there), and takes its clock format from its own configuration or the
  system defaults. Its accessibility menu (high contrast, large text,
  the screen reader, its own on-screen keyboard) is remembered by the
  greeter itself.

Each of these is written down in its contract, marked *GNOME does not
document this; Helix's choice:* where GNOME is silent, so other
implementations can follow or differ knowingly.

## Not yet verified

Some supported features are written to their published standards and
tested against simulated devices or services, but not yet with the real
thing. Each contract says so where it applies, and the full list is the
helixdesktop/desktop issues labelled `needs-hardware`; the main ones are touchscreens and touchpad gestures,
tearing (asynchronous page flips) for full-screen games,
drawing tablets, HDR and calibrated monitors, DDC/CI brightness, privacy
screens, underscanning, offline updates, user switching with a real
display manager, Bluetooth file transfer and USB protection. The
greeter has been verified on real hardware through greetd. Screen-reader support is tested over
AT-SPI on a private bus and was read with Orca 50 on a headless Helix
session (private buses, speech read from Orca's log; what it said is in
`contracts/dbus/org.a11y.atspi.md`, "Verified with Orca"); The file
chooser's accessible tree is tested over AT-SPI but has not been read
with Orca yet (helixdesktop/desktop#71). Orca's mouse
review does not work yet (the compositor does not tell where the
pointer is), and Orca has not been used on real hardware with speech.
