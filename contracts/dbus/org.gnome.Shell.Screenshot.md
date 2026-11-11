# org.gnome.Shell.Screenshot — behavioral notes

Companion to `org.gnome.Shell.Screenshot.xml`.

## Bus name and object

- Bus name: `org.gnome.Shell` (served by the shell)
- Object: `/org/gnome/Shell/Screenshot`

## Who may call it

Capturing the screen silently is restricted. Only callers that own one of
these well-known names may use the interface; others get
`org.freedesktop.DBus.Error.AccessDenied`:

- `org.freedesktop.impl.portal.desktop.helix` (Helix's portal backend,
  which asks the user or checks the app's permission first)
- `org.gnome.Screenshot` (the GNOME Screenshot app)

GNOME restricts this interface too, but which callers it allows is not part
of the contract (see `gaps.md`); the list above is Helix's.

## File names

`filename` (in `Screenshot`, `ScreenshotWindow` and `ScreenshotArea`) is
either an absolute path or a base name. A base name is saved in
`$XDG_PICTURES_DIR`, or in the home directory if that directory does not
exist. The image is always PNG; Helix adds `.png` when the name has no
extension of that kind, and never overwrites a file: if the name is taken,
it adds ` (2)`, ` (3)`, … before the extension. `filename_used` is the
absolute path written.

## Captures

All coordinates are logical (layout) pixels, the coordinate space of
`org.gnome.Mutter.DisplayConfig`. Images have as many pixels per logical
pixel as the highest scale among the monitors they cover; parts shown on a
monitor with a lower scale are scaled up.

- `Screenshot(include_cursor, flash, filename)`: every monitor, one image
  covering the bounding box of the layout. Parts of the box no monitor
  shows are transparent.
- `ScreenshotWindow(include_frame, include_cursor, flash, filename)`: the
  focused window, cropped to its window geometry (client-side shadows are
  left out). With `include_frame` the title bar Helix draws for X11
  windows is included (see `contracts/wayland/README.md`, "Window
  frames"). Open popup menus show where they lie inside the window. Fails
  (`success` false) when no window has focus.
- `ScreenshotArea(x, y, width, height, flash, filename)`: that rectangle.
  `width` and `height` must be positive, otherwise the call fails with
  `org.freedesktop.DBus.Error.InvalidArgs`.

`include_cursor` draws the pointer into the image when it is inside the
captured part. `flash` briefly whitens the captured area on screen after
the capture. The calls return `success` false (and an empty
`filename_used`) when the capture or the file could not be written, and
while the session is locked.

## Interactive methods

These show the shell's own UI and return when the user is done. Only one
runs at a time; a second call while one is open fails with
`org.freedesktop.DBus.Error.Failed`. When the user cancels (Escape, or the
secondary button) the call fails with `org.freedesktop.DBus.Error.Failed`
and the message `Operation was cancelled`.

- `SelectArea()`: the user drags out a rectangle on one monitor; returns
  it in logical layout coordinates.
- `PickColor()`: the user clicks a pixel; returns `{"color": (r, g, b)}`,
  sRGB values from 0 to 1.
- `InteractiveScreenshot()`: shows the screenshot UI (below) and returns
  `(true, uri)` with the `file://` URI of the saved image, or
  `(false, "")` when the user cancels.
- `FlashArea(x, y, width, height)`: flashes the rectangle; `width` and
  `height` must be positive.

## Screenshot UI (Helix)

Shown by `InteractiveScreenshot` and by the `show-screenshot-ui` key
(`contracts/gsettings/shell-keys.md`). It dims every monitor and shows a
hint:

- Dragging with the primary button selects an area on that monitor;
  releasing the button captures it.
- Enter or Space captures the whole screen.
- Escape or the secondary button cancels.

The UI is not part of the capture. Images are saved as
`Screenshot From YYYY-MM-DD HH-MM-SS.png` (local time) in a `Screenshots`
folder inside the pictures directory (as for base names above), which is
created if needed.

## After a screenshot the user took (Helix)

For screenshots taken in the screenshot UI or with the screenshot keys
(not for the other methods, which apps call):

- The image is put on the clipboard as `image/png`, replacing what was
  there, so it can be pasted at once (X11 apps included).
- A notification says "Screenshot captured" with the body "You can paste
  the image from the clipboard." Clicking it opens the image with the
  default application for PNG images; its "Show in Files" button shows the
  image in the file manager (`org.freedesktop.FileManager1`
  `ShowItems`), or opens the `Screenshots` folder when there is no file
  manager service.
