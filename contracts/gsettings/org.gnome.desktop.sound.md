# `org.gnome.desktop.sound`

Key surface of the `org.gnome.desktop.sound` GSettings schema in gsettings-desktop-schemas
51.0, described through GSettings introspection of the compiled schema.

Path: `/org/gnome/desktop/sound/`

| Key | Type | Default | Allowed values |
|-----|------|---------|----------------|
| `allow-volume-above-100-percent` | `b` | `false` |  |
| `event-sounds` | `b` | `true` |  |
| `input-feedback-sounds` | `b` | `false` |  |
| `theme-name` | `s` | `'freedesktop'` |  |

## Helix

### System sounds volume

The volume of system (event) sounds is not a key here: GNOME Settings
keeps it in the sound server, as PulseAudio's stream-restore entry
`sink-input-by-media-role:event` (libpulse's ext-stream-restore). Under
PipeWire, pipewire-pulse stores that entry in the `route-settings`
metadata object under `restore.stream.Output/Audio.media.role:Notification`
(it calls the `event` role `Notification`), and WirePlumber keeps it and
applies it to each new event sound. Helix's Settings app reads, follows
and writes that metadata key directly (with `pw-metadata`), so the volume
is shared with GNOME Settings and pavucontrol.

*GNOME does not document this; Helix's choice:* the value Helix writes is
the one pipewire-pulse writes for libpulse, `{ "mute": …, "volumes": [ v,
v ], "channels": [ "FL", "FR" ] }` with linear amplitudes (v is the
slider's level cubed, PulseAudio's volume curve), keeping the mute flag
as it was. It is stereo rather than mono because WirePlumber applies the
volumes by channel position, and a mono entry leaves the right channel of
a stereo event sound silent. Any layout written by others is read as its
loudest channel; no entry means 100 %. Event sounds playing at that
moment are changed too, as PulseAudio's stream-restore does. With
`event-sounds` off the volume is kept but has nothing to apply to.
Without the metadata object (no WirePlumber, or PulseAudio itself) Helix
offers no system sounds volume.
