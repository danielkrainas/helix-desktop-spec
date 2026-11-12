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
  A `.desktop` file shows its `Name` and `Icon`.
- A press selects the file under the pointer (an accent-colored rounded
  cell), or nothing. A second press on the same file within 400 ms opens
  it: a `.desktop` file of `Type=Application` that is executable starts
  its application (through `gio launch`), anything else opens with its
  default application.
- There are no menus, renaming, dragging or keyboard use; trusted
  launcher marks (`metadata::trusted`) are not used.

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
`secondary-color` and `color-shading-type`), dimmed by half under the clock
and password field.
