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
- `show-desktop-icons` has no effect.

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
