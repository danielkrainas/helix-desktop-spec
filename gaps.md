# Gaps

Living log of missing, unclear, or underspecified contracts.

## Implemented but not tested with real hardware

Features below are written to their published standards and checked with
unit and integration tests against simulated devices, but nobody has yet
tried them with the real hardware. Remove an entry once it has been.

- DDC/CI brightness of external monitors (`contracts/dbus/org.gnome.Mutter.DisplayConfig.md`, "Backlight"): tested against a simulated MCCS monitor only.
- Bluetooth file sending and receiving (`contracts/dbus/org.bluez.obex.md`): tested against a fake obexd only.
- Touchscreen and touchpad gestures (`contracts/wayland/README.md`, "Gestures"): unit tests, and touchscreen swipes through a remote desktop virtual touchscreen in a nested session; no real touchscreen or touchpad has been tried.
- HDR monitors (`org.gnome.Mutter.DisplayConfig.md`, "Color modes"): EDID parsing, KMS metadata and the PQ conversion are checked by unit tests and the conversion pass in a nested session; no HDR monitor has been tried.
- Color profiles (`org.gnome.Mutter.DisplayConfig.md`, "Color profiles"): tested against a fake colord and made-up profiles; no calibrated monitor has been tried.
- The eject key (`contracts/gsettings/shell-keys.md`, "Eject and radios"): the drive choice is unit tested; no drive was ejected (the rfkill keys were tried against fake NetworkManager and BlueZ services).
- Drawing tablets (`contracts/gsettings/input-settings.md`, "Tablets"): the tablet protocol, pressure curves, pads and their settings are tested in-process and by unit tests; no tablet, stylus or pad has been tried. Stylus `keybinding` actions share the pad buttons' key typing and are unit-tested only. The eraser button setup needs libinput 1.29, which the development machine does not have (1.25): only the run-time lookup (absent there) is tested; configuring a real tool has never run.

## Open gaps

- `org.gnome.Shell.Screenshot`: which callers GNOME allows (Helix allows its portal backend and `org.gnome.Screenshot`), the D-Bus error for a cancelled `SelectArea` / `PickColor` (Helix: `org.freedesktop.DBus.Error.Failed`), what happens when the file name is taken (Helix adds ` (2)` etc.), and the pixel scale of captures spanning monitors with different scales (Helix uses the highest) are undocumented. GNOME's screenshot UI is not documented here; Helix's (modes, toggles, keys) is its own design, in `org.gnome.Shell.Screenshot.md`.
- Screenshot portal: where GNOME's backend saves non-interactive screenshots, and when it asks the user, are undocumented. Helix asks (shows its screenshot UI) unless the frontend says the app's permission was checked, and saves in the `Screenshots` folder. Helix's window pick is its own design.
- Screen casting: which callers mutter lets use `org.gnome.Mutter.ScreenCast`, and its stream formats and frame pacing, are not documented here; Helix's are in `org.gnome.Mutter.ScreenCast.md`. Helix records monitors, windows, areas (spanning monitors too) and fixed-size virtual monitors (1920×1080; not negotiated with the consumer, not in DisplayConfig), remembers shared windows by app id and title for `persist_mode`. The contents of GNOME's `restore_data` are its own; Helix's are in `org.freedesktop.impl.portal.md`.
- DisplayConfig: D-Bus error names for rejected `ApplyMonitorsConfig` calls (Helix uses `org.freedesktop.DBus.Error.InvalidArgs` for stale serials and invalid configurations); per-monitor `properties` keys in `ApplyMonitorsConfig` (Helix ignores them); how GNOME's own confirmation of persistent changes is triggered (it happens inside gnome-shell; Helix uses its own `org.helix.DisplayChangeDialog`) and whether GNOME also asks after temporary changes; which fractional scales GNOME offers for a mode, and its preferred scale (Helix: 0.25 steps that give whole logical sizes; preferred 1); the meaning of `SetOutputCTM`'s nine values (Helix takes them as the kernel's `drm_color_ctm` and sets the CRTC's `CTM` property), and whether GNOME keeps gamma ramps across mode changes; whether GNOME's `Backlight` serial changes when a value changes, and the `min` it reports for kernel backlight devices (Helix: serial changes only with the set of entries; `min` 1). Helix does not implement `ChangeBacklight` (deprecated). GNOME's `min` for DDC/CI monitors is unknown (Helix: 0).
- Which input counts as "user activity" for IdleMonitor (e.g. whether pointer motion without buttons does). Helix counts every input event.
- Session startup ordering in GNOME 51 (gnome-session's systemd targets) is not documented here. Helix uses its own simple launcher; components announce readiness by owning a bus name or calling `SessionManager.Setenv` (the compositor sets `WAYLAND_DISPLAY`).
- Touchscreens and tablets: the touchscreen, tablet and stylus schemas are relocatable, and the per-device paths GNOME keeps them at are not documented; Helix's paths (by USB ids and stylus serial) are in `contracts/gsettings/input-settings.md`, "Touchscreens and tablets". How mutter identifies monitors (its vendor/product/serial strings from EDID) and maps tablets automatically is not documented; Helix's choices are in `org.gnome.Mutter.DisplayConfig.md` ("Monitor identity") and `input-settings.md`. Where GNOME keeps pad button settings (Helix: `button<letter>/` below the tablet's path), what its pad help display looks like, how it maps pad rings and strips to actions (Helix sends them to apps), and how it applies `pressure-curve` exactly are not documented here; Helix's choices are in `input-settings.md`, "Tablets". Which button GNOME has libinput send for an eraser button in `button` mode, and what `eraser-button-action` `default` means there, are not documented here; Helix's choices (the third stylus button; back) are in `input-settings.md`, "Tablets". GNOME's touch and touchpad gestures (which finger counts, distances and edges it uses, and what each swipe does) are not documented here; Helix's are in `contracts/wayland/README.md`, "Gestures". Helix takes an IBus engine's layout from IBus component files only (not from engines components list by running a program); how GNOME treats an engine whose layout is `default` is not documented (Helix keeps the last `xkb` layout). GNOME's IBus integration details (how its menus show engine properties and submenus, auxiliary text, emoji/Unicode modes) are not covered; Helix handles commit, pre-edit, candidates and properties (menus flattened into the input source menu). How GNOME maps `custom-accel-config` to libinput's acceleration types and X button numbers to evdev buttons is not documented here; Helix's mapping is in `contracts/gsettings/input-settings.md`.
- SessionManager D-Bus error names are unspecified. Helix uses `org.gnome.SessionManager.Error.{NotInInitialization,AlreadyInitialized,NotInRunning,NotRegistered,InvalidCookie,InvalidArgs,Inhibited,General}` pending a contract.
- SessionManager when the session bus goes away, or the session manager dies: gnome-session's behavior (and systemd's part in it) is not documented here; Helix ends the session and stops its components (`org.gnome.SessionManager.md`, "When the bus goes away").
- SessionManager inhibitor/client lifetime: Helix drops a caller's clients and inhibitors when its bus connection goes away (so a crashed app can't block logout forever). Not specified by the contract.
- SessionManager session restore: how gnome-session decides between reasons, when it relaunches apps, and where it keeps its records are not documented; Helix's choices are in `org.gnome.SessionManager.md`. Whether xdg-desktop-portal uses the API under `XDG_CURRENT_DESKTOP=Helix:GNOME` is untested.
- `org.gtk.Notifications` `ActionInvoked` signal (added in GNOME 51) is undocumented in gnome-shell; Helix follows the documented meaning of the same signal in xdg-desktop-portal's Notification backend (sent for actions that are not `app.` actions), see `org.gtk.Notifications.md`.
- Notifications: how gnome-shell keeps notifications after their banner (its message list and how it groups and stacks an app's notifications, limits per app, what it does with `expire_timeout` and the `transient` hint), and where it keeps per-application settings (`org.gnome.desktop.notifications.application`, listed in `application-children`; the child paths are undocumented), are not documented here. Helix's choices are in `org.freedesktop.Notifications.md` ("Helix"), including how it names the per-application settings' paths.
- `org.gnome.SessionManager.EndSessionDialog` `Open` type values beyond 0 (logout), 1 (power off) and 2 (restart) are undocumented (GNOME also has upgrade-related dialogs). Helix only uses 0–2.
- Portal backends: Helix implements Settings, Inhibit, Screenshot, ScreenCast, RemoteDesktop, Clipboard, Access, Account, AppChooser, Background, Wallpaper, Lockdown, Notification (version 2), Print, FileChooser, Email, DynamicLauncher, InputCapture (version 2, sharing the clipboard like RemoteDesktop) and GlobalShortcuts (version 1); `helix-portals.conf` sends any other interface to the GNOME and GTK backends when installed. Mutter's input capture interface (`org.gnome.Mutter.InputCapture`) and how GNOME treats barriers, keys held at activation and escaping a capture are not documented here; Helix uses its own `org.helix.InputCapture` (Helix-private) and its choices are in `org.freedesktop.impl.portal.md`. How GNOME's backends pick the mail app and its arguments (GTK's Email backend) and which apps GNOME's DynamicLauncher backend gives install tokens to are not documented here. GNOME's dialogs for Access, Account, AppChooser, Background and Wallpaper (their wording, AppChooser's search and "find more apps", Background's notifications, where its Wallpaper backend keeps copies, its print dialog's options and print-to-file location, its file chooser: GTK's, with places, search, previews and recent files; Helix's has places, recent files, a name search, GVfs network places and picture previews, but no previews of other files and no content search) and what GNOME does with the lockdown keys beyond the portal are not documented here; Helix's are in `contracts/dbus/org.freedesktop.impl.portal.md`. Helix reports only apps with windows to Background. What else GNOME's backend does beyond the documented interfaces is not covered here.
- Settings portal: namespaces beyond `org.freedesktop.appearance` are undocumented, yet GTK reads `org.gnome.desktop.interface` (and others) through the portal when sandboxed. Which namespaces and keys a GNOME-compatible backend must expose is not in the spec; Helix's set is in `org.freedesktop.impl.portal.md`. GNOME's own mapping from `accent-color` names to sRGB is not documented either; Helix uses the colors libadwaita documents for `AdwAccentColor`.
- Notification portal version 2: how GNOME shows categories and display hints, which sound `default` means, and which button purposes it offers are not documented here; Helix's choices are in `org.freedesktop.impl.portal.md`, "Notification".
- GlobalShortcuts portal: how GNOME's backend asks the user and lets them change triggers, where it keeps its answers, and the unit of the signals' `timestamp` are not documented here; Helix's choices are in `org.freedesktop.impl.portal.md`, "GlobalShortcuts" (milliseconds, as the shell reports them).
- Settings / Session portal backends: the value of the `version` property each backend should report is undocumented. Helix reports 1.
- Shell extension API surface (explicitly deferred; internal and unstable).
- `org.gnome.Shell` `GrabAccelerator`: the interface does not document `grabFlags`; Helix takes them to be mutter's documented `Meta.KeyBindingFlags` (as `Meta.Display.grab_accelerator` uses), see `org.gnome.Shell.md`.
- Window switching: GNOME's exact grouping and ordering rules for `switch-applications` (e.g. whether activating an application raises all its windows, and moving into an application's windows from the application switcher) are not documented here; Helix's rules are in `contracts/wayland/README.md`. GNOME's `cycle-windows` / `cycle-group` follow the held modifier, and `switch-panels` shows a popup of system controls; Helix's versions (time-based cycles, keyboard focus on its panel) are in `contracts/wayland/README.md`.
- Window frames: how GNOME's X11 frames look (size, colors, buttons) and which windows it frames beyond the Motif hints is not documented here; Helix's frames are in `contracts/wayland/README.md` ("Window frames"). How mutter shades windows (which windows, the `_NET_WM_STATE_SHADED` state, input and focus while shaded) is not documented here; Helix's shading is in `contracts/wayland/README.md`, "Window frames".
- Window menu: what GNOME's window menu lists and how its keyboard Move/Resize behave (e.g. arrow keys) is not documented here; Helix's menu and keys are in `contracts/wayland/README.md` ("Window menu and window keys"). Helix's Move/Resize keys are its own choice (arrows by 10 pixels, 1 with Control).
- Window-management settings: mutter's exact rules for `auto-maximize` (how close to the work area a window must be), `focus-change-on-pointer-rest` (the rest time), modifier resizes (which edge or corner) and where `strict` new windows go are not documented here; Helix's choices are in `contracts/wayland/README.md` ("Window-management settings"). `disable-workarounds` has no effect.
- Unresponsive apps: when mutter checks that an app is alive, its dialog's wording, and how GNOME ends an app on Force Quit are not documented here; Helix's choices are in `contracts/wayland/README.md` ("Unresponsive apps").
- Remote desktop: mutter's `Version`, how it turns keysyms into keys, and how its EIS server describes devices (names, regions) are not documented here; Helix's choices are in `org.gnome.Mutter.RemoteDesktop.md`. GNOME's `restore_data` contents are its own; Helix's are in `org.freedesktop.impl.portal.md`. (Clipboard sharing and keymaps: see "Remote desktop clipboard and keymaps" below.)
- Wayland protocols: mutter's built-in list of applications allowed to grab the keyboard under Xwayland (the schema only names a placeholder), and the exact handling of its X11 ClientMessage asking to grab, are not documented here; Helix's list and handling are in `contracts/wayland/README.md`, "Xwayland keyboard grabs". The permission store table and id GNOME uses for shortcut inhibition (`gnome` / `shortcuts-inhibitor`) are not documented in the schema or D-Bus contracts; Helix uses them so GNOME Settings can show and revoke the answers. Helix centers non-square `_NET_WM_ICON` images on a square, and switches virtual terminals with Ctrl+Alt+F1…F12 regardless of `switch-to-session-*`.
- Remote desktop clipboard and keymaps: GNOME's interface docs leave out
  errors, the format of `CurrentKeymap`'s `name`, how long a
  `SelectionTransfer` may go unanswered and whether `SelectionOwnerChanged`
  is emitted on `EnableClipboard`; Helix's choices are in
  `contracts/dbus/org.gnome.Mutter.RemoteDesktop.md`. XKB text format
  version 2 keymaps are not accepted; an EIS keyboard is not told about
  layout index changes (only keymap changes, by replacement).
- Color management and HDR: Helix converts only from sRGB (apps cannot
  send BT.2100, scRGB or ICC-described content; no per-surface
  conversion), keeps 8 bits per channel for PQ output, and does no tone
  mapping. GNOME's reference white, the `output-luminance` range it
  allows (Helix: SDR 10–100 % through the gamma ramps; BT.2100 25–300 %
  of 203 cd/m² as SDR white), the order of primaries in its HDR metadata
  (Helix: red, green, blue), how it derives the metadata's values, its
  colord device ids and properties, and how it combines profile
  calibration with night light are not documented here; Helix's choices
  are in `org.gnome.Mutter.DisplayConfig.md` ("Color modes", "Color
  profiles", "Gamma"). The color mode values (0 default, 1 BT.2100)
  follow GNOME's monitor settings; their DisplayConfig names are as GNOME
  Settings uses them.
- Screen recording: which callers GNOME lets use `org.gnome.Shell.Screencast`,
  its `Error` names, and what it does when a file exists are not documented;
  Helix's choices are in `contracts/dbus/org.gnome.Shell.Screencast.md`. How
  GNOME lays out monitors of different scales in one recording is not
  documented; Helix uses the highest scale.
- Keyboard accessibility: how GNOME implements the AccessX-style features
  (which keys count as modifiers for sticky keys, the keyboard shortcuts
  `enable` turns on, whether mouse keys need Num Lock off, beep sounds,
  and what the timeout turns off) is not documented by the schema; Helix's
  choices are in `contracts/gsettings/a11y-keyboard.md`. GNOME's dwell click type window (its look, place, icons and whether a
  type stays chosen) and progress feedback, and the exact formulas of its magnifier color effects and its
  high-contrast styling, are not documented here; Helix's are in
  `a11y-mouse.md`, `a11y-magnifier.md` and `a11y-interface.md`.
- Magnifier: GNOME's zoom steps for `magnifier-zoom-in` / `-out`, how
  `proportional` and `push` tracking move the view exactly, and the
  crosshairs' clip gap are not documented; Helix's choices are in
  `contracts/gsettings/a11y-magnifier.md`. GNOME's lens size and
  frame, and how it picks the point it follows for focus and caret, are
  not documented; Helix's are in `a11y-magnifier.md`.
- Autostart: gnome-session's autostart phases, delays, restarts and its
  full list of `AutostartCondition` forms are not documented here;
  Helix's subset is in `org.gnome.SessionManager.md` ("Autostart").
- Hot corner: GNOME triggers it by pointer pressure against a barrier;
  how much pressure and its timing are not documented. Helix's (100
  unaccelerated pixels within a second; reaching the corner for absolute
  pointers) is in `contracts/wayland/README.md`, "Overview".
- Polkit agent: what gnome-shell's dialog shows and how it lets the user
  pick among identities (a choice of users in the dialog; Helix asks
  first, in a dialog of its own) are not documented here, nor is
  polkit's helper protocol in polkit's own documentation (it is private to
  libpolkit-agent); Helix's choices are in
  `org.freedesktop.PolicyKit1.AuthenticationAgent.md`.
- Keyring prompts: gcr's prompter interface and secret exchange are internal
  to gcr (not a stable API, documented only in its sources); how
  gnome-shell's prompt looks and rates password strength is not documented
  here. Helix's choices are in `org.gnome.keyring.internal.Prompter.md`.
- Background: how GNOME blends slideshow transitions, which `<size>` it
  picks, how it scales `wallpaper`/`spanned` across monitors of different
  scales, and what its lock screen shows (blur, dimming) are not
  documented here; Helix's choices are in `contracts/gsettings/background.md`.
- Desktop icons: GNOME's shell itself draws none; Ubuntu ships an
  extension (Desktop Icons NG, with its own settings) whose behavior is
  documented only by that extension. Helix shows a simple set when
  `show-desktop-icons` is on (`contracts/gsettings/background.md`,
  "Desktop icons") and does not read the extension's settings.
- System status: how gnome-shell picks network and battery icons (its signal
  strength thresholds) and what its airplane mode switch turns off (it goes
  through gnome-settings-daemon's rfkill handling) are not documented here;
  Helix's choices are in `contracts/wayland/README.md` ("Quick settings",
  "System status"). Helix lists networks and paired devices. Its NetworkManager secret agent answers Wi-Fi WPA/SAE, WEP and LEAP passwords, 802.1X user names and passwords, mobile broadband passwords and SIM PINs, PPPoE user names and passwords and VPN secrets (through the VPN plugins' auth dialogs; their protocol is documented only by the plugins and was observed from OpenVPN's), WireGuard private and preshared keys, but not certificates, and stores no agent-owned secrets; gnome-shell's agent dialogs, its keyring use and how it lays out many VPN fields are not documented here. The agent identifier `org.gnome.Shell.NetworkAgent` is the one GNOME's shell is known to use.
- Search providers: how many results gnome-shell shows per provider, when it
  uses subsearches, its timeouts and how it shows results (grid or list) are
  not documented here; Helix's choices are in
  `org.gnome.Shell.SearchProvider2.md`.
- Animations: GNOME's animation curves, lengths and which surfaces it animates (and how it animates maximizing, full screen and tiling; Helix's way is its own) are not documented here; Helix's are in `contracts/wayland/README.md`, "Animations".
- Dock: vanilla GNOME has no dock; Helix's follows Ubuntu's Dash to Dock settings, whose behavior (its animation curves, what exactly `height-fraction` measures, and the look of its previews and their hover popups; Helix shows previews on click, has no hover popups, and shows quicklists as entries of the dock menu, without submenus) is documented only by that extension; Helix's subset is in `contracts/wayland/README.md` ("Dock").
- Favorites and app folders: GNOME's app grid (pages, `app-picker-layout`,
  dragging apps into folders) and dash (reordering by dragging, its context
  menus) are not documented here; Helix shows favorites in a dash and the
  launcher's list, and does not use `app-picker-layout`
  (`contracts/wayland/README.md`, "Favorites and app folders").
- Automatic rotation: when mutter rotates (it may limit it to tablet mode or
  to touchscreen devices) and how it moves other monitors is not documented
  here; Helix's choices are in `org.gnome.Mutter.DisplayConfig.md`
  ("Automatic rotation").
- Power management: gnome-settings-daemon's dim timing, its idle actions'
  details, its battery notifications' wording and thresholds and its lid
  handling are not documented here; Helix's choices are in
  `contracts/gsettings/power.md`. How mutter lays out monitors when the
  lid closes or opens (it keeps per-lid-state configurations) is not
  documented here; Helix's simpler rule is in the same file.
- Input source indicator: how gnome-shell abbreviates IBus engines (it uses
  the engine's symbol) and what else its menu offers (showing the layout)
  are not documented here; Helix's are in `contracts/gsettings/input-settings.md`.
- Window placement across monitors: mutter's rules (where new windows open, what happens to windows of a monitor that goes away) are not documented here; Helix's are in `contracts/wayland/README.md` ("Monitors and windows").
- Workspaces: GNOME's exact rules for dynamic workspaces (e.g. when an emptied workspace is removed, focus after switching) and for `workspaces-only-on-primary` are not documented here; Helix's are in `contracts/gsettings/workspaces.md`. Helix lays workspaces out in a row (as GNOME 51 does), so the `-up`/`-down` workspace keys do nothing; its overview (pictures renewed twice a second) and workspace switch popup are its own design (GNOME's is not documented here).
- Night Light: how GNOME turns a color temperature into gamma ramps, how it fades, and when it re-checks the schedule are undocumented; Helix's choices are in `contracts/gsettings/night-light.md`. The behavior of gnome-settings-daemon's `org.gnome.SettingsDaemon.Color` (what writing `Temperature` does, how long `DisabledUntilTomorrow` lasts, the preview's limits, `Sunrise`/`Sunset` without a location) is undocumented; Helix's is in `org.gnome.SettingsDaemon.Color.md`.
- Brightness and volume keys: GNOME's brightness step, which monitor the `screen-brightness-*-monitor` keys mean by "current monitor", the step of the `-precise` volume keys and the OSD icon thresholds are undocumented. Helix: brightness steps of 1/20 of the range on every controllable monitor (the `-monitor` keys act the same), precise steps of 1 percent, icons switching at 33 and 66 percent (`contracts/gsettings/shell-keys.md`). Playback keys go to MPRIS players (Helix's choice of player is in `org.mpris.MediaPlayer2.Player.md`). GNOME's seek step for `playback-rewind`/`-forward`, its text size step, its keyboard backlight step and what its `power` key does by default (`org.gnome.settings-daemon.plugins.power`) are not documented; Helix's choices are in `contracts/gsettings/shell-keys.md`. What GNOME's `eject` key ejects and its rfkill on-screen displays are not documented here; Helix's choices are in `contracts/gsettings/shell-keys.md`, "Eject and radios".
- Removable media: gnome-shell's automount and autorun rules (which volumes it mounts at login, its notification's wording and buttons, the app chooser it shows for `autorun-x-content-start-app` and unhandled content types, running a medium's autorun programs) are not documented here; Helix's are in `contracts/gsettings/removable-media.md`. Helix asks the questions of its own mounts and ejects (`contracts/gsettings/removable-media.md`, "Questions") and GTK apps' through `org.gtk.MountOperationHandler`, whose interface is documented only in GTK and gnome-shell (its argument types were observed from GTK's calls; `contracts/dbus/org.gtk.MountOperationHandler.md`). How gnome-shell lays out these dialogs, which `response_details` keys it returns beyond GTK's, how it treats repeated calls and `Close`, and how it names busy apps (by their windows) are not documented here. Helix offers no anonymous login and no VeraCrypt hidden-volume, system-volume or PIM fields.
- Location: how gnome-shell's GeoClue agent asks (its dialog, whether it asks per accuracy level or remembers per session), how it stores decisions in the permission store (the value format of table `location`), and when it shows its location indicator are not documented here; Helix's choices are in `contracts/dbus/org.freedesktop.GeoClue2.Agent.md`. Helix registers with GeoClue as `gnome-shell` because GeoClue only accepts agents its configuration lists; a distribution that lists another id for Helix would need that id instead.
- Privacy indicators: which audio streams gnome-shell counts as using the microphone (e.g. how it treats level meters and recordings of outputs), how it tells cameras from other video sources, and how its indicators look and what they list are not documented here; Helix's choices are in `contracts/wayland/README.md` ("System status").
- Message tray calendar: gnome-shell reads events through its own helper (`gnome-shell-calendar-server`, a private interface) and expands recurrences with libical; Helix reads EDS directly (`contracts/dbus/org.gnome.evolution.dataserver.md`) and expands the rule parts listed in `contracts/gsettings/calendar.md` (not `SECONDLY` or `EXRULE`, and `BYHOUR`/`BYMINUTE`/`BYSECOND` only narrow hourly and minutely rules); how libical treats a start that the rule would not pick and times that fall in a daylight saving gap is not documented here. gnome-shell's tray layout, its event list's wording, how it finds a world clock's time zone (GWeather's location database) and its media controls are not documented here; Helix's choices are in `contracts/gsettings/calendar.md`. Helix finds a world clock's time zone from the nearest `zone1970.tab` city, which can be wrong near zone borders.
- Wellbeing: what gnome-shell does with the break-reminder and screen-time settings (how it counts active time, what its break screen and notifications say and offer, how `delay-seconds` and the overdue reminders work, where and how it keeps screen time history, how it turns the screen gray) is not documented here; Helix's choices are in `contracts/gsettings/wellbeing.md`. GNOME's history file is not read or written by Helix.
- On-screen keyboard: gnome-shell's keyboard (its layouts, which come from its own data files, its symbol and emoji pages, word suggestions, when it shows for touch versus pointer input, and how it moves windows out of the way) is not documented here; Helix's is in `contracts/gsettings/screen-keyboard.md`. Helix's keyboard takes its letters, AltGr page and currency sign from the layout, its accents from one table for all languages, and its suggestions from hunspell dictionaries; GNOME's per-language symbol pages, its word-prediction engine and whether it learns typed words are not documented here. Helix keeps typed words in memory only.
- Bluetooth pairing: how gnome-shell and Settings word and lay out their pairing dialogs, which agent capability they register with, and how long discovery runs are not documented here; Helix's choices are in `contracts/dbus/org.bluez.Agent1.md` and `contracts/wayland/README.md` ("Wi-Fi networks and Bluetooth devices"). Helix does not offer Bluetooth device settings. For OBEX file transfers, GNOME's wording, when it accepts pushes (and from whom), and where it puts received files before they complete are not documented here; Helix's choices are in `contracts/dbus/org.bluez.obex.md`.
- Weather: gnome-shell gets weather through libgweather (its providers, caching, and how `automatic-location` finds the place) and its weather section's look is not documented here; Helix asks MET Norway directly and names an automatic location after the nearest time zone city (`contracts/gsettings/calendar.md`, "Weather"). The `org.gnome.shell.weather` key table was taken from gnome-shell 46.
- X11 scaling and XSETTINGS: how mutter picks the automatic Xwayland scale (`xwayland-scaling-factor` 0) and what gnome-settings-daemon's XSETTINGS plugin publishes (its full list, its DPI rules, whether it merges or replaces `RESOURCE_MANAGER`) are not documented here; Helix's choices are in `contracts/wayland/README.md` ("X11 apps: scaling and settings"). Helix scales Xwayland through the compositor (it does not use Xwayland's own HiDPI mode).
- Accessibility menu: which switches gnome-shell's accessibility menu holds in GNOME 51, their order and the text scale it uses for Large Text are not documented here; Helix's are in `contracts/wayland/README.md` ("Accessibility menu").
