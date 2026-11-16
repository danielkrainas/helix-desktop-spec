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

*GNOME does not document this; Helix's choice:* GNOME restricts this
interface too, but which callers it allows is not part of the contract;
the list above is Helix's.

## File names

`filename` (in `Screenshot`, `ScreenshotWindow` and `ScreenshotArea`) is
either an absolute path or a base name. A base name is saved in
`$XDG_PICTURES_DIR`, or in the home directory if that directory does not
exist. The image is always PNG; Helix adds `.png` when the name has no
extension of that kind, and never overwrites a file: if the name is taken,
it adds ` (2)`, ` (3)`, … before the extension. `filename_used` is the
absolute path written.

*GNOME does not document this; Helix's choice:* what happens when the name
is taken (the ` (2)` suffix above).

## Captures

All coordinates are logical (layout) pixels, the coordinate space of
`org.gnome.Mutter.DisplayConfig`. Images have as many pixels per logical
pixel as the highest scale among the monitors they cover; parts shown on a
monitor with a lower scale are scaled up. (*GNOME does not document
this; Helix's choice:* the highest scale.)

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
and the message `Operation was cancelled`. (*GNOME does not document
this; Helix's choice:* the error for a cancelled call.)

- `SelectArea()`: the user drags out a rectangle on one monitor; returns
  it in logical layout coordinates.
- `PickColor()`: the user clicks a pixel; returns `{"color": (r, g, b)}`,
  sRGB values from 0 to 1.

**Keyboard.** *GNOME does not document this; Helix's choice:* the area
and point (and window) pickers work without a pointer. The first arrow
key makes the area (the middle quarter of the first monitor: half its
width and height, centered) or the point (the first monitor's middle);
the arrow keys then move it 10 logical px (with Shift 50, with Alt 1),
kept on its monitor; Ctrl with the arrows resizes the area, moving its
right and bottom edges (Right and Down grow it, Left and Up shrink it,
at least 10 px a side). Enter or Space picks the area or point; Escape
cancels. The area is shown as a dragged one (undimmed, outlined, its
size in a pill), the point as a small outlined box. The hints say so:
"Drag, or arrows to move and Ctrl+arrows to resize · Enter to confirm ·
Esc to cancel", "Click, or arrows and Enter, to pick a color · Esc to
cancel", "Click a window, or arrows and Enter, to capture it · Esc to
cancel". On the accessibility bus the hint is announced when the picker
opens and the keys' choice as it changes
(`contracts/dbus/org.a11y.atspi.md`, "Picking an area or a point").
- `InteractiveScreenshot()`: shows the screenshot UI (below) and returns
  `(true, uri)` with the `file://` URI of the saved image, or
  `(false, "")` when the user cancels.
- `FlashArea(x, y, width, height)`: flashes the rectangle; `width` and
  `height` must be positive.

## Screenshot UI (Helix)

Shown by `InteractiveScreenshot` and by the `show-screenshot-ui` and
`show-screen-recording-ui` keys (`contracts/gsettings/shell-keys.md`). It
covers every monitor; a toolbar, a rounded panel 40 logical px above the
bottom of the primary monitor, holds from the left round-cornered icon
toggles in groups (Helix's own symbols: a selection's corners, a monitor,
a window; a camera, a video camera; an arrow), the toggle that is on in
the accent fill, then a large round capture button (a white disc in a
ring; red while Record is chosen) and a round close button. Under the
pointer a toggle is lit (Hovered; pressed: Pressed) and a tooltip above
the toolbar names it and its key ("Area (S)", "Capture (Enter)", "Close
(Esc)"); its key is also its description on the accessibility bus. A
click acts on release over the button (`components/README.md`, "Hover
and press"). The buttons:

- three modes, one chosen (S, C, W):
  - **Area** (the start): dragging with the primary button selects a
    rectangle on one monitor, which is shown undimmed with the rest
    dimmed, outlined in the accent with eight round white handles (its
    corners and the middles of its edges) and its size in pixels (the
    monitor's, "1280 × 720") in a small pill under it (above it at the
    monitor's bottom). Dragging a handle moves those sides (past the
    opposite side the rectangle flips); dragging inside it moves it,
    kept on its monitor; dragging elsewhere selects a new one;
  - **Screen**: every monitor, nothing dimmed;
  - **Window**: the window under the pointer (topmost of the active
    workspace's windows) is outlined; clicking one selects it;

  in Area mode the arrow keys make, move and (with Ctrl) resize the
  rectangle as in the area picker (above); in Window mode they move a
  point as in the point picker, and the window under it is selected;
- **Screenshot** / **Record** (V switches): what Capture does. Record is
  offered only from the keys, not to `InteractiveScreenshot`; Window
  cannot be recorded (choosing Record leaves Window for Area);
- **Show Pointer** (P): whether the pointer is in the image or recording
  (off at the start);
- **Capture** (or **Start Recording**), also Enter or Space: takes the
  screenshot of the selection, or starts recording it
  (`contracts/dbus/org.gnome.Shell.Screencast.md`). It does nothing in
  Area mode before a rectangle is selected, or in Window mode before a
  window is.
- **Close**: cancels, as Escape.

No hint line is shown (the keys are in the tooltips). Escape or the
secondary button cancels. The UI is not part of the
capture. Images are saved as
`Screenshot From YYYY-MM-DD HH-MM-SS.png` (local time) in a `Screenshots`
folder inside the pictures directory (as for base names above), which is
created if needed.

## Picking a window (Helix)

Helix-private, for its Screenshot portal backend: interface
`org.helix.ScreenshotWindowPicker` at the same object, with the same
callers allowed. `PickWindow(include_cursor, flash, filename)` dims every
monitor with the hint "Click a window, or arrows and Enter, to capture
it · Esc to cancel" (the keys as for `PickColor`, above).
Clicking a window captures it as `ScreenshotWindow` with `include_frame`
true does, and returns `(success, filename_used)`; clicking where there is
no window shows the hint again. Escape or the secondary button cancels:
the call fails with `org.freedesktop.DBus.Error.Failed` and the message
"Operation was cancelled" (as `SelectArea`).

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
