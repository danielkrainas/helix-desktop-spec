# Background (Helix)

How Helix shows `org.gnome.desktop.background`
(`org.gnome.desktop.background.md`) behind the windows, and the lock
screen's picture from `org.gnome.desktop.screensaver`
(`org.gnome.desktop.screensaver.md`). Changes apply at once.

## The picture

- `picture-uri` (a `file://` URI or an absolute path), or `picture-uri-dark`
  while `org.gnome.desktop.interface` `color-scheme` is `prefer-dark` (and
  it is set).
- Formats: PNG, JPEG, WebP, JPEG XL and SVG (by content, whatever the file
  is called), and slideshows (below). A picture that cannot be read is
  left out (only the colors show).
- `picture-options`:
  - `none`: no picture, only the colors;
  - `wallpaper`: tiled at its own size from the monitor's top-left corner;
  - `centered`: its own size, centered;
  - `scaled`: as large as fits whole, centered;
  - `stretched`: stretched to the monitor;
  - `zoom`: as small as covers the monitor, centered and cropped;
  - `spanned`: as `zoom`, over the bounding box of all monitors (each
    monitor showing its part).
  Sizes are in logical pixels; the picture is drawn at each monitor's
  scale.
- `picture-opacity`: the picture over the colors, 0 to 100 percent.
- The colors under the picture: `primary-color` (`#rrggbb`), or with
  `color-shading-type` `vertical` / `horizontal` a gradient from
  `primary-color` (top / left) to `secondary-color`.
- `show-desktop-icons`: see "Desktop icons" below.

## Desktop icons

With `org.gnome.desktop.background show-desktop-icons` true, the shell
shows the files of the desktop folder (XDG `DESKTOP`; nothing when that
is the home folder itself) on the primary monitor, on a see-through
surface just above the background (layer-shell `bottom` layer,
namespace `helix-background-icons`, clear of the panel and a fixed
dock).

- Which files: all but hidden ones (`.name`) and backups (`name~`);
  folders first, then by name ignoring case. The folder is looked at
  again every two seconds while icons are shown.
- Each takes a 96 × 96 cell (columns filled top to bottom, from the
  left, 12 pixels from the edges; files that do not fit are not shown):
  a 48-pixel icon and, under it, its name on up to two lines (cut short
  with "…"), white with a shadow. Folders use `folder`, other files the
  icon of the content type their name suggests (then `text-x-generic`).
  A launcher shows its `Name` and `Icon`: a `.desktop` file of
  `Type=Application` that is executable, not writable by others (as
  Desktop Icons NG) and trusted (`metadata::trusted` is `true`, as
  "Allow Launching" below, GNOME's file manager's, or `gio set FILE
  metadata::trusted true` leaves it). Any other `.desktop` file is a
  plain file, under its file name: archives keep the executable bit, and
  the name and icon are whatever the file says.
- A press selects the file under the pointer (an accent-colored rounded
  cell), or nothing. A second press on the same file within 400 ms opens
  it: a launcher (checked again then) starts its application (through
  `gio launch`), anything else opens with its default application.
- A right click or long press on an empty spot (no icon) opens the
  background menu (`contracts/wayland/README.md`, "Background menu").
- **A file's menu**, as Desktop Icons NG's: a right press on an icon
  (opened on the press, at the pointer), or a long press on it (the
  primary button or a touch held 500 ms without moving more than 8
  logical pixels, as the background menu), selects it and opens its menu;
  so do Menu and Shift+F10 on the selected icon while the icons have the
  keyboard (at the icon's middle, its first entry selected). It lists:
  - **Open**: as a double click.
  - a separator and **Allow Launching** or **Don't Allow Launching**,
    only for an application's `.desktop` file (`Type=Application`, with a
    `Name`) right in the desktop folder that is a regular file (not a
    link) owned by the user and not writable by others. "Don't Allow
    Launching" when it launches now (as above), else "Allow Launching".
    Both look at the file again when chosen. "Allow Launching" sets
    `metadata::trusted` to `true` and, when the owner cannot execute it,
    adds the owner's executable bit (as Desktop Icons NG); "Don't Allow
    Launching" sets `metadata::trusted` to `false` and leaves the mode
    alone (who set the executable bit is not known). The icon shows the
    launcher's name and icon, or its own again, at once.
  Other Desktop Icons NG entries (Open With, Cut/Copy, Rename, Move to
  Trash, Properties, Show in Files…) are not offered.
- **Keyboard**: the icons surface asks for `on_demand` keyboard
  interactivity, so a click on it gives it the keyboard
  (`contracts/wayland/README.md`, "Shell surfaces and the keyboard").
  While it has it: Menu or Shift+F10 opens the selected file's menu,
  Return or Enter opens the selected file, Escape takes the selection
  away. There is no moving between icons with the keyboard, renaming or
  dragging.
- GNOME's shell draws no desktop icons; Ubuntu's Desktop Icons NG
  extension does, with settings of its own, which Helix does not read.

## Slideshows

A picture file whose root element is `<background>` (GNOME's background
XML) is a slideshow:

- `<starttime>` (`<year>`, `<month>`, `<day>`, `<hour>`, `<minute>`,
  `<second>`, local time) starts it; then come, in order, `<static>`
  (`<duration>` seconds, `<file>`) and `<transition>` (`<duration>`,
  `<from>`, `<to>`) parts, repeated in a loop after the last.
- A `<file>` may hold `<size width="…" height="…">path</size>` elements
  instead of a path; the one closest to the monitor's pixel size is used.
- During a static part its file shows; during a transition, `from` shows
  for the first half and `to` for the second. The picture changes when
  the part does.

## Lock screen

The lock screen shows `org.gnome.desktop.screensaver` `picture-uri` the
same way (with its `picture-options`, `picture-opacity`, `primary-color`,
`secondary-color` and `color-shading-type`), then blurred and toned so
that text on it stays readable:

- Blurred heavily (about a Gaussian of 24 logical pixels' deviation), with
  the colors' saturation slightly lowered (85% kept) and the whole picture
  slightly darkened.
- Toned: each pixel's luminance is capped, smoothly, lower in the middle
  column (where the clock, the user and the messages are) than at the
  left and right edges. Light text (#EDEDEF) on it has at least 4.5:1
  contrast everywhere, and the lock screen's secondary text (#C4C4CC) in
  the middle column, whatever the picture.
- It is made once per picture, monitor size and scale, not at each
  repaint. The login screen (`components/greeter.md`) can use the same
  treatment.
- In high contrast (`a11y-interface.md`) the lock screen's background is
  black instead.
