# Gaps

Living log of missing, unclear, or underspecified contracts.

## Open gaps

- `org.gnome.Shell.Screenshot`: which callers GNOME allows (Helix allows its portal backend and `org.gnome.Screenshot`), the D-Bus error for a cancelled `SelectArea` / `PickColor` (Helix: `org.freedesktop.DBus.Error.Failed`), what happens when the file name is taken (Helix adds ` (2)` etc.), and the pixel scale of captures spanning monitors with different scales (Helix uses the highest) are undocumented. GNOME's screenshot UI (window selection, screen recording, copying to the clipboard) is much richer than Helix's; Helix captures an area or the whole screen.
- Screenshot portal: where GNOME's backend saves non-interactive screenshots, and when it asks the user, are undocumented. Helix asks (shows its screenshot UI) unless the frontend says the app's permission was checked, and saves in the `Screenshots` folder. Helix's window pick is its own design.
- Screen casting: which callers mutter lets use `org.gnome.Mutter.ScreenCast`, and its stream formats and frame pacing, are not documented here; Helix's are in `org.gnome.Mutter.ScreenCast.md`. Helix records monitors, windows, areas (spanning monitors too) and fixed-size virtual monitors (1920×1080; not negotiated with the consumer, not in DisplayConfig), remembers shared windows by app id and title for `persist_mode`. The contents of GNOME's `restore_data` are its own; Helix's are in `org.freedesktop.impl.portal.md`.
- DisplayConfig: D-Bus error names for rejected `ApplyMonitorsConfig` calls (Helix uses `org.freedesktop.DBus.Error.InvalidArgs` for stale serials and invalid configurations); per-monitor `properties` keys in `ApplyMonitorsConfig` (Helix ignores them); how GNOME's own confirmation of persistent changes is triggered (it happens inside gnome-shell; Helix uses its own `org.helix.DisplayChangeDialog`) and whether GNOME also asks after temporary changes; which fractional scales GNOME offers for a mode, and its preferred scale (Helix: 0.25 steps that give whole logical sizes; preferred 1); the meaning of `SetOutputCTM`'s nine values (Helix takes them as the kernel's `drm_color_ctm` and sets the CRTC's `CTM` property), and whether GNOME keeps gamma ramps across mode changes; whether GNOME's `Backlight` serial changes when a value changes, and the `min` it reports for kernel backlight devices (Helix: serial changes only with the set of entries; `min` 1). Helix does not implement `ChangeBacklight` (deprecated) or DDC/CI backlights.
- Which input counts as "user activity" for IdleMonitor (e.g. whether pointer motion without buttons does). Helix counts every input event.
- Session startup ordering in GNOME 51 (gnome-session's systemd targets) is not documented here. Helix uses its own simple launcher; components announce readiness by owning a bus name or calling `SessionManager.Setenv` (the compositor sets `WAYLAND_DISPLAY`).
- Input settings Helix does not apply yet: the tablet schemas (no tablet input), and the touchscreen schema: `org.gnome.desktop.peripherals.touchscreen` is relocatable, and the per-device path mutter keeps it at is not documented, so Helix maps touchscreens automatically (built-in panel, else primary monitor) and has no touch gestures. Helix takes an IBus engine's layout from IBus component files only (not from engines components list by running a program); how GNOME treats an engine whose layout is `default` is not documented (Helix keeps the last `xkb` layout). GNOME's IBus integration details (properties menus, auxiliary text, emoji/Unicode modes) are not covered; Helix handles commit, pre-edit and candidates. How GNOME maps `custom-accel-config` to libinput's acceleration types and X button numbers to evdev buttons is not documented here; Helix's mapping is in `contracts/gsettings/input-settings.md`.
- SessionManager D-Bus error names are unspecified. Helix uses `org.gnome.SessionManager.Error.{NotInInitialization,AlreadyInitialized,NotInRunning,NotRegistered,InvalidCookie,InvalidArgs,Inhibited,General}` pending a contract.
- SessionManager inhibitor/client lifetime: Helix drops a caller's clients and inhibitors when its bus connection goes away (so a crashed app can't block logout forever). Not specified by the contract.
- SessionManager session restore: how gnome-session decides between reasons, when it relaunches apps, and where it keeps its records are not documented; Helix's choices are in `org.gnome.SessionManager.md`. Whether xdg-desktop-portal uses the API under `XDG_CURRENT_DESKTOP=Helix:GNOME` is untested.
- `org.gtk.Notifications` `ActionInvoked` signal (added in GNOME 51) is undocumented in gnome-shell; Helix follows the documented meaning of the same signal in xdg-desktop-portal's Notification backend (sent for actions that are not `app.` actions), see `org.gtk.Notifications.md`.
- Notifications: how gnome-shell keeps notifications after their banner (its message list, limits per app, what it does with `expire_timeout` and the `transient` hint), and where it keeps per-application settings (`org.gnome.desktop.notifications.application`, listed in `application-children`; the child paths are undocumented), are not documented here. Helix's choices are in `org.freedesktop.Notifications.md` ("Helix"), including how it names the per-application settings' paths.
- `org.gnome.SessionManager.EndSessionDialog` `Open` type values beyond 0 (logout), 1 (power off) and 2 (restart) are undocumented (GNOME also has upgrade-related dialogs). Helix only uses 0–2.
- Portal backends: Helix implements Settings, Inhibit, Screenshot, ScreenCast, RemoteDesktop and Clipboard; `helix-portals.conf` sends every other interface to the GNOME and GTK backends when installed. What else GNOME's backend does beyond the documented interfaces is not covered here.
- Settings portal: namespaces beyond `org.freedesktop.appearance` are undocumented, yet GTK reads `org.gnome.desktop.interface` (and others) through the portal when sandboxed. Which namespaces and keys a GNOME-compatible backend must expose is not in the spec; Helix's set is in `org.freedesktop.impl.portal.md`. GNOME's own mapping from `accent-color` names to sRGB is not documented either; Helix uses the colors libadwaita documents for `AdwAccentColor`.
- Settings / Session portal backends: the value of the `version` property each backend should report is undocumented. Helix reports 1.
- Shell extension API surface (explicitly deferred; internal and unstable).
- `org.gnome.Shell` `GrabAccelerator`: the interface does not document `grabFlags`; Helix takes them to be mutter's documented `Meta.KeyBindingFlags` (as `Meta.Display.grab_accelerator` uses), see `org.gnome.Shell.md`.
- Window switching: GNOME's exact grouping and ordering rules for `switch-applications` (e.g. whether activating an application raises all its windows, and moving into an application's windows from the application switcher) are not documented here; Helix's rules are in `contracts/wayland/README.md`. GNOME's `cycle-windows` / `cycle-group` follow the held modifier, and `switch-panels` shows a popup of system controls; Helix's versions (time-based cycles, keyboard focus on its panel) are in `contracts/wayland/README.md`.
- Window frames: how GNOME's X11 frames look (size, colors, buttons) and which windows it frames beyond the Motif hints is not documented here; Helix's frames are in `contracts/wayland/README.md` ("Window frames"). Helix does not shade windows.
- Window menu: what GNOME's window menu lists and how its keyboard Move/Resize behave (e.g. arrow keys) is not documented here; Helix's menu and keys are in `contracts/wayland/README.md` ("Window menu and window keys"). Helix's Move/Resize keys are its own choice (arrows by 10 pixels, 1 with Control).
- Window-management settings: mutter's exact rules for `auto-maximize` (how close to the work area a window must be), `focus-change-on-pointer-rest` (the rest time), modifier resizes (which edge or corner) and where `strict` new windows go are not documented here; Helix's choices are in `contracts/wayland/README.md` ("Window-management settings"). X11 bells (XKB) are not passed on; `disable-workarounds` has no effect.
- Unresponsive apps: when mutter checks that an app is alive, its dialog's wording, and how GNOME ends an app on Force Quit are not documented here; Helix's choices are in `contracts/wayland/README.md` ("Unresponsive apps").
- Remote desktop: mutter's `Version`, how it turns keysyms into keys, and how its EIS server describes devices (names, regions) are not documented here; Helix's choices are in `org.gnome.Mutter.RemoteDesktop.md`. GNOME's `restore_data` contents are its own; Helix's are in `org.freedesktop.impl.portal.md`. (Clipboard sharing and keymaps: see "Remote desktop clipboard and keymaps" below.)
- Wayland protocols: mutter builds a default list of applications allowed to grab the keyboard under Xwayland into itself (the schema only names a placeholder), and also honors an X11 ClientMessage asking to grab; Helix has no built-in list and ignores the ClientMessage. The permission store table and id GNOME uses for shortcut inhibition (`gnome` / `shortcuts-inhibitor`) are not documented in the schema or D-Bus contracts; Helix uses them so GNOME Settings can show and revoke the answers. Helix takes only square images from `_NET_WM_ICON` and does not implement `xwayland-scaling-factor` (it needs Xwayland's own scaling, from version 24.1), and switches virtual terminals with Ctrl+Alt+F1…F12 regardless of `switch-to-session-*`.
- Remote desktop clipboard and keymaps: GNOME's interface docs leave out
  errors, the format of `CurrentKeymap`'s `name`, how long a
  `SelectionTransfer` may go unanswered and whether `SelectionOwnerChanged`
  is emitted on `EnableClipboard`; Helix's choices are in
  `contracts/dbus/org.gnome.Mutter.RemoteDesktop.md`. XKB text format
  version 2 keymaps are not accepted; an EIS keyboard is not told about
  layout index changes (only keymap changes, by replacement).
- Color management and HDR: Helix does no color conversion and has no HDR
  output (no BT.2100 color mode, no KMS HDR metadata, no PQ encoding); its
  `wp_color_manager_v1` offers only sRGB. How GNOME applies
  `output-luminance` to an SDR monitor, and the range it allows, are not
  documented; Helix dims through the gamma ramps between 10 % and 100 %.
  The color mode values (0 default, 1 BT.2100) follow GNOME's monitor
  settings; their DisplayConfig names are as GNOME Settings uses them.
- Screen recording: which callers GNOME lets use `org.gnome.Shell.Screencast`,
  its `Error` names, and what it does when a file exists are not documented;
  Helix's choices are in `contracts/dbus/org.gnome.Shell.Screencast.md`. How
  GNOME lays out monitors of different scales in one recording is not
  documented; Helix uses the highest scale.
- Keyboard accessibility: how GNOME implements the AccessX-style features
  (which keys count as modifiers for sticky keys, the keyboard shortcuts
  `enable` turns on, whether mouse keys need Num Lock off, beep sounds,
  and what the timeout turns off) is not documented by the schema; Helix's
  choices are in `contracts/gsettings/a11y-keyboard.md`. Helix does not
  implement `org.gnome.desktop.a11y.mouse` or the universal access menu.
- Magnifier: GNOME's zoom steps for `magnifier-zoom-in` / `-out`, how
  `proportional` and `push` tracking move the view exactly, and the
  crosshairs' clip gap are not documented; Helix's choices are in
  `contracts/gsettings/a11y-magnifier.md`. Helix does not support the
  magnifier's color effects, lens mode, screen positions other than full
  screen, or focus and caret tracking.
- Autostart: gnome-session's autostart phases, delays, restarts and its
  full list of `AutostartCondition` forms are not documented here;
  Helix's subset is in `org.gnome.SessionManager.md` ("Autostart").
- Hot corner: GNOME triggers it by pointer pressure against a barrier (how
  much, and its timing, is not documented); Helix triggers on reaching the
  corner pixel (`contracts/wayland/README.md`, "Overview").
- Window placement across monitors: mutter's rules (where new windows open, what happens to windows of a monitor that goes away) are not documented here; Helix's are in `contracts/wayland/README.md` ("Monitors and windows").
- Workspaces: GNOME's exact rules for dynamic workspaces (e.g. when an emptied workspace is removed, focus after switching) and for `workspaces-only-on-primary` are not documented here; Helix's are in `contracts/gsettings/workspaces.md`. Helix lays workspaces out in a row (as GNOME 51 does), so the `-up`/`-down` workspace keys do nothing; its overview (pictures renewed twice a second) and workspace switch popup are its own design (GNOME's is not documented here).
- Night Light: how GNOME turns a color temperature into gamma ramps, how it fades, and when it re-checks the schedule are undocumented; Helix's choices are in `contracts/gsettings/night-light.md`. The behavior of gnome-settings-daemon's `org.gnome.SettingsDaemon.Color` (what writing `Temperature` does, how long `DisabledUntilTomorrow` lasts, the preview's limits, `Sunrise`/`Sunset` without a location) is undocumented; Helix's is in `org.gnome.SettingsDaemon.Color.md`.
- Brightness and volume keys: GNOME's brightness step, which monitor the `screen-brightness-*-monitor` keys mean by "current monitor", the step of the `-precise` volume keys and the OSD icon thresholds are undocumented. Helix: brightness steps of 1/20 of the range on every controllable monitor (the `-monitor` keys act the same), precise steps of 1 percent, icons switching at 33 and 66 percent (`contracts/gsettings/shell-keys.md`). Playback keys go to MPRIS players (Helix's choice of player is in `org.mpris.MediaPlayer2.Player.md`). GNOME's seek step for `playback-rewind`/`-forward`, its text size step, its keyboard backlight step and what its `power` key does by default (`org.gnome.settings-daemon.plugins.power`) are not documented; Helix's choices are in `contracts/gsettings/shell-keys.md`. Helix does not handle `eject`, `rfkill` or `rotate-video-lock`.
