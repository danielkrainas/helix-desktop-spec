# org.a11y.atspi — the shell on the accessibility bus

The shell draws its interface itself, so it publishes that interface on
the accessibility bus itself, as one AT-SPI application, the way a
toolkit does for an app's windows. A screen reader (Orca) reads it,
follows its focus and operates it like any other application's.

The AT-SPI interfaces (`org.a11y.atspi.Accessible`, `Component`,
`Action`, `Text`, `EditableText`, `Value`, `Selection`, `Cache`,
`Application`) and the registry's `org.a11y.atspi.Socket` are at-spi2-core's;
Helix does not restate them here. These notes say what the shell
publishes through them.

## Joining the bus

- The shell watches `org.a11y.Status` `IsEnabled` on the session bus
  (served by at-spi2-core's bus launcher at `org.a11y.Bus`
  `/org/a11y/bus`). While it is true, the shell connects to the
  accessibility bus (`AT_SPI_BUS_ADDRESS` if set, else `org.a11y.Bus`
  `GetAddress`), serves its root at `/org/a11y/atspi/accessible/root` and
  embeds it with the registry's `Socket.Embed`. When it turns false the
  shell leaves.
- Without a session bus, an `org.a11y.Bus`, or a registry, nothing is
  published and the shell works as before. `HELIX_NO_ATSPI` (any value)
  keeps the shell off the bus.
- Trees are only built for windows a client has asked for, so an
  unused bus costs nothing.

## The application

- Role `application`; its name is the shell's program name,
  `helix-shell` (GNOME Shell's is `gnome-shell`). Toolkit name `Helix`,
  toolkit version the shell's version.
- Each shell surface shown is one child of the application: a window
  (AT-SPI role `frame`), named as below, whose extents are the surface's
  place on the screen (layout coordinates, logical pixels). Its children's
  extents are their places on that surface.
- The window holding the keyboard is `active` and gets `window:activate`;
  focus inside it is the `focused` state on one descendant, with focus
  events as it moves.
- While the session is locked only the lock screen is published: one
  window, for the lock surface with the prompt (see
  `org.gnome.ScreenSaver.md`, "Monitors"), so there is a single password
  field; the other monitors show only the time.

| Window name | Surface | Has the keyboard |
|---|---|---|
| `Lock Screen` | the lock screen | always |
| the dialog's title | a dialog (end session, display change, not responding, shortcuts, remote control, extension approval, screen sharing, prompts) | always |
| `Windows`, `Applications` or `Input Sources` | the switcher (Alt+Tab and friends) | always |
| `Overview` | the launcher (search and app grid) | always |
| `System Menu` | quick settings | while open |
| `Notifications` | the notification list and calendar | while open |
| `Top Bar` | the panel | with `switch-panels` focus |
| `Notification` | a banner | with `focus-active-notification` |
| `Dock` | each monitor's dock | never |

## What each window holds

Names, states and places come from what the surface last painted, and
change when it does. Objects keep their identity across changes (a
changing clock is the same label; a toggled switch is the same switch).

- **Lock screen**: resting (before a key or click), labels for the clock
  and the date, the media player and notifications as below, and a
  focused `Unlock` button that shows the prompt. With the prompt: labels
  for the clock, the date and the user name; a
  `password text` named `Password`, focused (insensitive while the
  password is checked); the message under it (`Checking…`, `Incorrect
  password. Try again.`) as a label announced when it appears or
  changes; the media player's title and artist and its buttons (`Previous
  Track`, `Play`/`Pause`, `Next Track`); one label per app with
  notifications ("Mail: 2 notifications"); the `Restart…`, `Power Off…`
  and `Switch User…` buttons when shown, and when one was pressed the
  question (announced), the busy apps, `Cancel` (which has the focus
  until the keyboard moves it) and the confirming button. Buttons the
  keyboard focused (Tab) have the focus instead of the password field.
- **Question dialogs**: a modal `dialog` named by the title, the message
  as a label (or the header and a list of busy apps, each named by the
  app with its reason as description), the `Install pending software
  updates` check box when offered, and the two buttons. The action
  button has the focus (Enter answers with it).
- **Prompt dialogs** (polkit, keyring, network secrets, mount questions,
  portal dialogs, the run dialog): a modal `dialog` named by the title;
  the message and description as labels; places and list rows (`list
  item`s, selected as picked); the text field (`text`) and password
  fields (`password text`, named by their labels; a field turned to
  "Show" is a `text`); `Show Password` toggle buttons; the check boxes
  (`check box`, checked as set); the status line (`Authenticating…` or
  the warning) announced; and the cancel and continue buttons (continue
  insensitive while answering). Focus follows Tab among the fields and
  check boxes; with no field, the list's current row, else the continue
  button, has it.
- **Screen sharing**: a modal `dialog` `Share your screen?`, the message,
  one `list item` per monitor (connector as description; selected when
  picked, if several may be), `Cancel`, and `Share` when several may be.
- **Top bar**, left to right as placed: `Apps` (a toggle button, pressed
  while the launcher is open), `Workspace N of M` (when there are
  several), the clock (a button named by the time, expanded while the
  notification list is open; described "Notifications and calendar",
  plus ", unread notifications" while the dot shows), one button per
  status icon (named by its title, else its tooltip title; described by
  the tooltip text), `Accessibility`, the input source (named by its
  indicator), `System` (described by the status icons in words, e.g.
  "Wi-Fi Home, Volume muted, Battery 80%, charging, Microphone in use";
  expanded while quick settings is open), `Power`, extension items (their
  accessible name, else label, tooltip or name), `Sharing` (a label) and
  `Recording` (a button that stops it). With `switch-panels`, the
  control with the focus ring is focused.
- **Quick settings**: a `menu` named `System` holding the sliders
  (`slider` with the value and range shown: `Volume` 0–maximum percent,
  `Brightness` 0–100, extension sliders 0–100), `Mute` (a toggle
  button), the switches (`toggle button`, checked when on; extension
  toggles described by their subtitle), the buttons opening a switch's
  list (`Wi-Fi Networks`, `Bluetooth Devices`, expanded while open) or an
  extension's menu (`<name> Settings`), the open list's rows (menu items,
  described `Connected`, `Secured`), the power modes (`radio button`s in
  a `Power Mode` group, checked for the active one) and the power
  actions (menu items).
- **Switcher**: a `list box` of the entries shown, named by their
  labels with their detail as description; the selected one is selected
  and focused.
- **Overview**: a `Search` entry holding the query. With a query, a
  `Search Results` list box of the rows shown (named as shown, described
  by their tag: "Open window", "Folder", "Favorite", or a provider
  result's description and app), the selected row focused; with none,
  "No matching apps". Without a query, an `Applications` group of the
  page's apps and folders (buttons; the selected one selected and
  focused, else the entry is), the open folder's apps instead (a group
  named by the folder), and `Page N` buttons when there are several
  pages.
- **Banner**: a `notification` named by the summary, described by the
  body (markup removed), announced; the app name, `Close`, the action
  buttons by their labels and the body's links (`link`).
- **Notifications**: a `Notifications` list of notifications (list
  items named "App: summary", described by the body, each with its
  `Close` and links) and group headers ("App, N notifications", with
  `Clear`), "No Notifications" when empty; the calendar's `Previous
  Month`/`Next Month` and day buttons ("14 November 2026"), media
  buttons, `Clear`, and `Do Not Disturb` (a toggle button).
- **Dock**: a `tool bar` `Dock` of buttons: apps by their names
  (described `Running` with a window open), drives, extension items,
  `Trash`, and `Show Apps` (pressed while the launcher is open).

## Actions

Every object that reacts to clicks offers one action, `click`; doing it
does what a click at the middle of the object's place does (the same
code runs). Focusing a text field (`Component.GrabFocus`) clicks it too.
A slider's `Value.CurrentValue` (and increment/decrement by its step,
5%) clicks the bar where that value is. Fields' `EditableText` calls
change nothing: typing goes through the keyboard. Other actions are
ignored.

## Not covered

- Not yet published (helix-desktop#73): the window overview (window
  pictures and workspaces); menus the shell pops up (panel status icon
  and extension menus, the dock's and app grid's menus, the window menu,
  the input source and accessibility menus); the workspace popup; the
  on-screen keyboard; the dwell click type window; desktop icons; the
  screenshot UI and area picker; the break screen; "Customize Panel…";
  the Extensions dialog; the OSD; panel tooltips and monitor labels. The
  dock has no keyboard navigation, so it has no focus to follow. Text
  fields expose their text but no character extents, and `EditableText`
  changes nothing. The calendar's events are plain labels (no times or
  durations as values).
- Windows' screen positions are computed from the layout (the primary
  monitor, the dock's edge) rather than reported by the compositor.
- The tree, roles, names, states, focus and actions are tested over
  AT-SPI on a private accessibility bus, but not yet with Orca on a
  Helix session: whether Orca announces windows and focus changes as
  intended (activation of layer surfaces, live messages, the lock
  screen's password field) is unverified (helix-desktop#71).
