# Shell keys — screenshots, brightness and volume

How Helix applies the screenshot and brightness keys of
`org.gnome.shell.keybindings` and the volume and playback keys of
`org.gnome.settings-daemon.plugins.media-keys` (key tables in the files
named after each schema). In GNOME 51 the shell handles the first two
groups and gnome-settings-daemon the volume keys; in Helix the shell
handles all of them.

Each key holds a list of accelerators in GTK syntax (`<Shift>Print`);
empty strings are ignored. Installed schemas older than 51 may lack some
keys (the brightness keys moved into `org.gnome.shell.keybindings`); a
missing key uses the default from the 51.0 table.

## Screenshots

| Key | Action |
|-----|--------|
| `show-screenshot-ui` | show the screenshot UI (`contracts/dbus/org.gnome.Shell.Screenshot.md`) |
| `screenshot` | capture every monitor at once |
| `screenshot-window` | capture the focused window at once |

Captures taken at once flash the captured area and are saved like the
screenshot UI's: `Screenshot From YYYY-MM-DD HH-MM-SS.png` in the
`Screenshots` folder of the pictures directory. They do not work while
the session is locked.

`show-screen-recording-ui` opens the screenshot UI with Record chosen;
recordings are saved as `Screencast From %d %t.webm` in the videos
directory (`contracts/dbus/org.gnome.Shell.Screencast.md`). While one
runs, the key stops it.

## Overview and launcher

| Key | Action |
|-----|--------|
| `toggle-overview` | show or hide the overview (`contracts/wayland/README.md`, "Overview") |
| `toggle-application-view` | show or hide the application launcher |
| `shift-overview-up` | from the desktop to the overview, from the overview to the launcher |
| `shift-overview-down` | from the launcher to the overview, from the overview to the desktop |
| `toggle-message-tray` | show or hide the notification list |
| `focus-active-notification` | give the notification banner keyboard focus |

A focused banner is outlined: Enter activates it (as a click on it),
Delete dismisses it (as its close button), Escape gives focus back without
closing it. It still goes away when its time is up.

`toggle-quick-settings` opens or closes the quick settings menu
(`contracts/wayland/README.md`, "Quick settings").

## Brightness

| Key | Action |
|-----|--------|
| `screen-brightness-up` | one step brighter |
| `screen-brightness-down` | one step darker |
| `screen-brightness-cycle` | one step brighter; from the brightest, back to the darkest |

The keys change every monitor that has a controllable `Backlight` entry
(`org.gnome.Mutter.DisplayConfig.md`), through `SetBacklight`. A step is
1/20 of the entry's `min`…`max` range (at least 1). The `-monitor`
variants act like the plain keys in Helix (see `gaps.md`).

The shell then shows an on-screen display with the icon
`display-brightness-symbolic` and the new level, `(value - min) / (max -
min)`.

## Volume

Keys of `org.gnome.settings-daemon.plugins.media-keys`, each with a
`-static` twin (the same action; the static ones have the defaults such as
`XF86AudioRaiseVolume`):

| Key | Action |
|-----|--------|
| `volume-up`, `volume-down` | the default output one step louder / quieter |
| `volume-up-quiet`, `volume-down-quiet`, `volume-mute-quiet` | the same; Helix plays no feedback sounds anyway |
| `volume-up-precise`, `volume-down-precise` | one percent louder / quieter |
| `volume-mute` | mute or unmute the default output |
| `mic-mute` | mute or unmute the default input |

- A step is `volume-step` percent of full volume.
- The volume stays between 0 and 100 percent, or 150 percent when
  `org.gnome.desktop.sound` `allow-volume-above-100-percent` is true.
- Raising the volume of a muted output unmutes it.
- The shell shows an on-screen display: for the output, the icon
  `audio-volume-muted-symbolic` (muted or at 0),
  `audio-volume-low-symbolic` (up to 33 percent),
  `audio-volume-medium-symbolic` (up to 66 percent) or
  `audio-volume-high-symbolic`, and the level with `max_level` 1 (1.5
  when above 100 percent is allowed); for the input,
  `microphone-sensitivity-muted-symbolic` or
  `microphone-sensitivity-high-symbolic` and no level.

## Playback

Keys of `org.gnome.settings-daemon.plugins.media-keys`, each with a
`-static` twin holding the defaults (`XF86AudioPlay` and so on):

| Key | Call on the player |
|-----|--------------------|
| `play` | `PlayPause` |
| `pause` | `Pause` |
| `stop` | `Stop` |
| `next` | `Next` |
| `previous` | `Previous` |

| `playback-rewind` | `Seek` 10 seconds back |
| `playback-forward` | `Seek` 10 seconds ahead |
| `playback-repeat` | set `LoopStatus` to the next of `None`, `Playlist`, `Track` (then `None` again) |
| `playback-random` | flip `Shuffle` |

The player is chosen as in `contracts/dbus/org.mpris.MediaPlayer2.Player.md`.
Nothing happens when no player is running.

## Launching

Keys of `org.gnome.settings-daemon.plugins.media-keys`, each with a
`-static` twin:

| Key | Opens |
|-----|-------|
| `www` | the default web browser (the app for `x-scheme-handler/https`) |
| `email` | the default mail app (`x-scheme-handler/mailto`) |
| `calculator` | `org.gnome.Calculator`, else the first app in the `Calculator` category |
| `control-center` | `org.gnome.Settings` |
| `home` | the home folder, in the app for folders |
| `media` | the default music player (the app for `audio/x-vorbis+ogg`) |
| `help` | `help:gnome-help` |
| `search` | the application launcher (as `toggle-application-view`) |

Nothing happens when there is no such app.

## Session

| Key | Action |
|-----|--------|
| `screensaver` | lock the screen |
| `logout` | `org.gnome.SessionManager` `Logout(0)` (asks first) |
| `shutdown`, `power` | `Shutdown()` (asks first) |
| `reboot` | `Reboot()` (asks first) |
| `suspend` | `Suspend()` |
| `hibernate` | logind `Hibernate(false)`, when `CanHibernate` is `yes` |

## Text size, contrast, touchpad

| Key | Action |
|-----|--------|
| `increase-text-size`, `decrease-text-size` | `org.gnome.desktop.interface` `text-scaling-factor` up / down by 0.25, between 0.5 and 3 |
| `toggle-contrast` | flip `org.gnome.desktop.a11y.interface` `high-contrast` |
| `touchpad-on`, `touchpad-off`, `touchpad-toggle` | set `org.gnome.desktop.peripherals.touchpad` `send-events` to `enabled` / `disabled` / the other one; the shell shows `input-touchpad-symbolic` or `touchpad-disabled-symbolic` |

## Keyboard backlight and battery

Through UPower (`org.freedesktop.UPower`, system bus):

- `keyboard-brightness-up`, `-down`: `KbdBacklight` `SetBrightness` one
  step (a tenth of `GetMaxBrightness`, at least 1) up or down;
  `keyboard-brightness-toggle`: to 0, or from 0 back to the last other
  value (else the maximum). The shell shows `keyboard-brightness-symbolic`
  and the level.
- `battery-status`: the shell shows the display device's `IconName` and
  `Percentage` (nothing without a battery).

## Custom shortcuts

`custom-keybindings` lists object paths; each holds a
`org.gnome.settings-daemon.plugins.media-keys.custom-keybinding` (relocatable
schema: `name`, `command`, `binding`). Pressing `binding` runs `command`,
split into arguments with shell quoting rules but without a shell, in the
shell's environment. Entries with an empty `binding` or `command` do
nothing.

## Changes

Changes to the keys of `org.gnome.shell.keybindings`,
`org.gnome.settings-daemon.plugins.media-keys` (including custom shortcuts)
and `org.gnome.desktop.wm.keybindings` apply at once.

## Not supported

`rotate-video-lock` (each with a `-static` twin) flips
`org.gnome.settings-daemon.peripherals.touchscreen` `orientation-lock` (see
`org.gnome.Mutter.DisplayConfig.md`, "Automatic rotation"); the shell shows
`rotation-locked-symbolic` "Rotation Locked" or `rotation-allowed-symbolic`
"Auto Rotate". `eject`, `rfkill` and `rfkill-bluetooth` do nothing.
GNOME 51 has no `org.gnome.SettingsDaemon.MediaKeys` D-Bus interface (apps
grabbing the media keys); players are controlled through MPRIS only.

## Accessibility

`magnifier`, `magnifier-zoom-in`, `magnifier-zoom-out`, `screenreader` and
`on-screen-keyboard` (also `org.gnome.settings-daemon.plugins.media-keys`)
are described in `a11y-magnifier.md`.

## While locked

Volume, brightness, keyboard backlight, playback, accessibility,
`suspend`, `hibernate` and `battery-status` keys keep working while the
session is locked (the on-screen display is not shown on the lock screen). Screenshot
keys do not.
