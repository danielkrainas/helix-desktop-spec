# org.mpris.MediaPlayer2.Player — what Helix relies on

Companion to `org.mpris.MediaPlayer2.Player.xml`, the part of the MPRIS 2.2
specification Helix's shell uses for the playback keys
(`contracts/gsettings/shell-keys.md`). Media players implement it; Helix
calls it.

## Finding players

Each player owns a bus name starting with `org.mpris.MediaPlayer2.` and
serves the interface at `/org/mpris/MediaPlayer2`.

## Members

- `PlaybackStatus` (`s`): `Playing`, `Paused` or `Stopped`.
- `PlayPause`, `Play`, `Pause`, `Stop`, `Next`, `Previous`: what they
  say. A player ignores a call it cannot honor (`CanPlay`, `CanPause`,
  `CanGoNext`, `CanGoPrevious` false); with `CanControl` false it does not
  accept any of them.

## Which player the keys control (Helix)

The first player, in bus-name order, whose `PlaybackStatus` is `Playing`;
else the first that is `Paused`; else the first one found. Players with
`CanControl` false are skipped.
