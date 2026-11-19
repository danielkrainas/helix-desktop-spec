# org.gnome.Shell.Screencast — behavioral notes

Companion to `org.gnome.Shell.Screencast.xml`: recording the screen into a
video file.

## Bus name and object

- Bus name: `org.gnome.Shell.Screencast`
- Object: `/org/gnome/Shell/Screencast`

## Methods

- `Screencast(file_template, options)` records the whole screen,
  `ScreencastArea(x, y, width, height, file_template, options)` a
  rectangle (layout coordinates, logical pixels). Both return `(success,
  filename_used)`.
- `file_template`: a file name, absolute or relative; `%d` becomes the
  start date and `%t` the start time. A relative name is placed in the
  videos directory (`XDG_VIDEOS_DIR`), or the home directory when there is
  none.
- `options`: `draw-cursor` (`b`, default true), `framerate` (`i`, default
  30), `pipeline` (`s`, a GStreamer pipeline in `gst-launch` syntax that
  encodes the frames; default: VP8 video in WebM).
- `StopScreencast()` ends the recording; `success` is false when there was
  none.
- `ScreencastSupported` (`b`): whether recording works here.
- `Error(name, message)` reports a recording that failed after it started.

## Helix

*GNOME does not document this; Helix's choice:* which callers may record,
the `Error` names, what happens when the file exists, and the scale of a
recording spanning monitors of different scales are as below.

- The shell serves it. The only caller is Helix's portal backend (the
  owner of `org.freedesktop.impl.portal.desktop.helix`, not in an app
  sandbox), besides the shell itself; others, `org.gnome.Screenshot`
  included, get `org.freedesktop.DBus.Error.AccessDenied`. (GNOME too
  lets only the shell record.)
- `pipeline` may only chain encoders and the elements around them:
  `videoconvert`, `videoscale`, `videorate`, `videoflip`, `queue`, the
  encoders `vp8enc`, `vp9enc`, `x264enc`, `x265enc`, `openh264enc`,
  `av1enc`, `svtav1enc`, `rav1enc`, `theoraenc`, `vaapih264enc`,
  `vah264enc`, `vah265enc`, `vaav1enc`, `nvh264enc`, `nvh265enc`, and the
  parsers `h264parse`, `h265parse`, `vp9parse`, `av1parse`, each with
  plain `property=value` settings (not `name` or `location`), linked by
  `!`, and `video/x-raw,…` caps. Anything else (other elements, element
  names and references, branches, bins, quoting) is
  `org.freedesktop.DBus.Error.InvalidArgs`.
- `%d` becomes `YYYY-MM-DD` and `%t` `HH-MM-SS` (local time). A name
  without an extension gets `.webm`, or `.mkv` with a `pipeline`. An
  existing file is not replaced: ` (2)`, ` (3)`… are added before the
  extension.
- One recording at a time: a second `Screencast` / `ScreencastArea`
  returns `(false, "")`, as does an empty area or one on no monitor.
- The whole screen is every monitor (virtual ones included): the bounding
  box of the layout, at
  the highest monitor scale (parts on no monitor show the background
  color). An area is recorded as `org.gnome.Mutter.ScreenCast`
  `RecordArea` records it (it may span monitors). `draw-cursor` true embeds the
  pointer, false leaves it out.
- Frames come from an `org.gnome.Mutter.ScreenCast` stream and are encoded
  by GStreamer (`gst-launch-1.0`): `pipewiresrc`, the frame rate capped at
  `framerate`, then `videoconvert` and, by default, `vp8enc` into `webmmux`
  (with `pipeline`, the given elements and `matroskamux`).
  `ScreencastSupported` is true when `gst-launch-1.0` and the default
  pipeline's elements are installed. `StopScreencast` lets GStreamer finish
  the file (end of stream) before returning, for at most 5 seconds.
- When GStreamer stops on its own, the recording ends and `Error` is
  emitted with the name `org.gnome.Shell.Screencast.Error.Failed` and
  GStreamer's last error message (or "the recording stopped"); an empty
  file is removed.
- The panel shows a "Recording" indicator while a recording runs; clicking
  it stops the recording. When a recording is stopped, a notification
  "Screencast recorded" names the file (Open, Show in Files).
