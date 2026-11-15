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
| `Launcher` | the launcher (search and app grid) | always |
| `System Menu` | quick settings | while open |
| `Notifications` | the notification list and calendar | while open |
| `Top Bar` | the panel | with `switch-panels` focus |
| `Notification` | a banner | with `focus-active-notification` |
| `Dock` | each monitor's dock | with `switch-panels` focus |
| `On-screen display` | the OSD while shown | never |
| `Window Overview` | the window overview, one window per monitor (the primary monitor's first) | the primary monitor's, always |
| the menu's name (below) | a shell menu | always |
| `Workspace Switcher` | the workspace popup | never |
| `On-screen Keyboard` | the on-screen keyboard | never (it types into the focused app) |
| `Click Type` | the dwell click type window | never |
| `Desktop` | the desktop icons | never |
| `Screenshot` | the screenshot UI and the area, point and window pickers (the primary monitor's surface) | always |
| the break's title | the break screen | with the keyboard |
| `Customize Panel` | "Customize Panel…" | always |
| `Extensions` | the Extensions dialog | always |
| `Tooltip` | a panel tooltip | never |
| `Monitor N` | each monitor's label (`ShowMonitorLabels`) | never |

Window names are translated. *GNOME does not document its window names;
Helix's choice:* the launcher and the window overview have names of
their own (`Launcher`, `Window Overview`), distinct from each other
and from the switcher's, so a screen reader user (or a script looking a
window up by name) can tell which one is open.

Which window has the keyboard follows the order keys go in: the lock
screen, a menu, the screenshot UI, dialogs, the switcher, the overview,
the break screen, "Customize Panel…", the Extensions dialog, the
launcher, quick settings, the notification list, the panel.

## What each window holds

Names, states and places come from what the surface last painted, and
change when it does. Objects keep their identity across changes (a
changing clock is the same label; a toggled switch is the same switch).
Names, descriptions and announcements are in the user's language: the
same translated strings the surface shows (`components/i18n.md`). On a
surface mirrored for a right-to-left language, places are those of its
left-to-right layout; actions still reach the control under them.

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
  while the launcher is open, described `Workspace N of M`; a
  `Workspaces` button alike when only `helix:workspaces` is shown), the clock (a button named by the time, expanded while the
  notification list is open; described "Notifications and calendar",
  plus ", unread notifications" while the dot shows), one button per
  status icon (named by its title, else its tooltip title; described by
  the tooltip text), `Accessibility`, the input source (named by its
  indicator), `System` (described by the status icons in words, e.g.
  "Wi-Fi Home, Volume muted, Battery 80%, charging, Microphone in use";
  expanded while quick settings is open; the status icons and the power
  icon are this one button), extension items (their
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
- **Launcher**: a `Search` entry holding the query. With a query, a
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
- **Dock**: a `tool bar` `Dock` of buttons: apps by their names,
  drives, extension items, `Trash`, and `Show Apps` (pressed while the
  launcher is open). Each item's description says what the dock paints
  on it, in this order, comma-separated: its windows ("1 window", "3
  windows"; at most four are counted, as there are at most four marks),
  its badge ("4 new"), its progress bar ("45 % done") and wanting
  attention ("needs attention"); empty when none applies. The item of
  the focused window's app (or drive, or Trash) is `selected`. As the
  dock is drawn again when any of these change, the change reaches
  clients as AT-SPI events (`object:property-change:accessible-description`,
  `object:state-changed:selected`), so a screen reader can announce
  "Mail, 1 window, needs attention".
  *GNOME does not document this; Helix's choice:* the words and their
  order; AT-SPI's `active` state, which GNOME's dash does not use for
  this either, is not available to the shell's AT-SPI layer, so the
  focused app's item is `selected` instead.

- **Window overview**: a `Search` entry (it opens the launcher when
  typed into); a `Workspaces` list box of the workspace thumbnails, named
  by `workspace-names` ("Workspace 2" otherwise), the active one
  selected; a `Windows` list box of the window pictures, each named by
  its window's title and described by its app's name ("No open windows"
  when empty), the chosen one selected; and a `Dash` tool bar of app
  buttons (described "Running" when the app has windows). The keyboard's
  place has the focus, with focus events as Tab moves between the parts
  and the arrows inside them: the chosen window, the workspace or dash
  icon the arrows reached, else the entry. Other monitors' overviews have
  their `Windows` list only.
  *GNOME does not document this; Helix's choice:* window pictures are
  list items of a list box (selection is the chosen window), workspace
  thumbnails list items whose selection is the active workspace.
- **Menus** (panel status icon and extension item menus, the dock's and
  app grid's app menus, the window menu, the input source and
  accessibility menus, the panel's context menu): a `menu` named by what
  it is for (the status icon or extension item, the app, the drive or
  `Trash`, `Input Sources`, `Accessibility`, `Top Bar`, `Window Menu`) of
  its entries where painted, with `separator`s between groups. Entries
  with a check box are `check menu item`s, checked as set; a set of
  choices (input sources, engine modes) are `radio menu item`s, the
  current one checked; an entry with a submenu is expandable (its
  submenu replaces the menu, with `Back` first); disabled entries are
  insensitive. The selected entry (keyboard or pointer) is selected and
  has the focus, with a focus event at each move; with none, the menu has
  the focus. A dock app's window previews are a `menu` (named `Windows`)
  of each window's entry, named by its title, and its `Close`, described
  by the title.
- **Workspace popup**: the workspace switched to, as a live `status bar`
  (announced), and a `Workspaces` list of the workspaces, the active one
  selected.
- **On-screen keyboard**: the suggestion bar (`Suggestions`, or `Accents`
  for a letter's accents) as a tool bar of buttons, then a `Keyboard`
  panel of the keys where painted. Character keys are buttons named by
  the character they type (shifted when Shift is latched); Shift and the
  page keys (`Symbols`, `Emoji`, `AltGr`) are toggle buttons, pressed while
  latched or on their page; `Backspace`, `Enter`, `Tab`, `Space`, `Left`,
  `Right` and `Hide Keyboard` are buttons.
  *GNOME does not document this; Helix's choice:* the key names and the
  page keys as toggle buttons; the window never takes the keyboard, so it
  is never active and has no focus.
- **Click type window**: a `Click Type` panel of four `radio button`s
  (`Single`, `Double`, `Drag`, `Secondary`), the current type checked.
- **Desktop icons**: a `Desktop` list box of the items in their cells,
  named by their names, described `Folder` or `Application` when they are
  one; the selected one selected. A click selects; clicking the selected
  item again opens it, as a double click does.
- **Screenshot UI**: the hint as a label, the area or window to be
  captured as a label "Selected: W × H" at its place, and a `Screenshot`
  tool bar: `Area`, `Screen`, `Window`, and (when recording is allowed)
  `Screenshot` and `Record` as `radio button`s checked as chosen (`Window`
  insensitive while recording), `Show Pointer` a toggle button, and
  `Capture` (`Start Recording`). The area, point and window pickers show
  the hint only: choosing a point or an area needs the pointer.
- **Break screen**: a modal `dialog` named by the break's title, its
  message (announced), the time left as a `timer` ("4:59") and the
  `Delay` and `Skip` buttons.
- **Customize Panel…**: a modal `dialog` with a `list` per column (`Left`,
  `Center`, `Right`, `Quick Settings`) of its items (list items) holding
  their buttons (`Move Up`, `Move Down`, `Move Left`, `Move Right`,
  `Remove`, each described by the item's name); a `Hidden` list of the
  hidden items with `Add`; `Reset` and `Done`. The button Tab reached has
  the focus.
- **Extensions dialog**: a modal `dialog` with a list of the extensions in
  view, each a list item named by its name and version, described by its
  description, state and what it may do; holding its on/off switch (a
  toggle button named by the extension, pressed when on), `Settings`
  (expanded while open), its commands with their keys (labels) and, when
  open, its settings: switches, choices (buttons described by the value),
  numbers (`Decrease` and `Increase`, described "Size: 3") and text
  (labels). "Scroll for more" when some are out of view; `Done`. The
  control Tab reached has the focus.
- **OSD**: a live `status bar` named by what it shows ("Volume 40%");
  each change reaches clients as a name change and is announced.
- **Panel tooltips**: a `tool tip` named by the tooltip's first line and
  described by the rest. The panel button it belongs to is described by
  the same text.
- **Monitor labels**: a label with the monitor's number, described by its
  connector.
- **Calendar events**: the selected day's events are list items of the
  `Events` list, each named by its summary and described by when it is
  and how long it lasts: "10:00 to 11:30, 1 hour 30 minutes", "10:00"
  (no end), "All Day", "All day, 3 days".
  *GNOME does not document this; Helix's choice:* the times and the
  length are the description, in the user's clock format.

## Text fields

The shell's text fields (the launcher's search, the run dialog, the
prompts' text fields, password fields) offer `Text` and `EditableText`.

- `Text` holds what the field shows: the typed text, or for a password
  field one `•` per character (`password text` role); the password itself
  is never published. A field turned to "Show" is a `text` holding its
  text.
- `GetCharacterExtents` and `GetRangeExtents` give each character's box
  as painted: a letter's advance on the line, a password dot's box. A
  field that scrolls keeps the characters before its start to its left.
- `SetTextContents` replaces the field's text. `InsertText` and
  `DeleteText` change it in place; on a password field the change is made
  to the password at the same offsets. Control characters are dropped. A
  field that is checking (the lock screen while the password is
  verified, a prompt that was answered) ignores changes.
  *GNOME does not document this; Helix's choice:* edits never submit the
  field (Enter, or the action of the continue button, does).
- `CopyText` puts the characters in the range on the clipboard,
  `CutText` does that and removes them, `PasteText` inserts the
  clipboard's text at the offset (line breaks become spaces, other
  control characters are dropped). A password field takes a paste but
  is never copied or cut: `CopyText` and `CutText` on it change nothing.
  It is the clipboard the keys use (`contracts/wayland/README.md`,
  "Shell text fields and the clipboard").

## Actions

Every object that reacts to clicks offers one action, `click`; doing it
does what a click at the middle of the object's place does (the same
code runs). Focusing a text field (`Component.GrabFocus`) clicks it too.
A slider's `Value.CurrentValue` (and increment/decrement by its step,
5%) clicks the bar where that value is. Fields' `EditableText` calls
are described under "Text fields". Other actions are ignored.

## Not covered

- Pickers that need the pointer (choosing an area or a point for a
  screenshot, dragging windows between workspaces in the overview) have
  no accessible way to do it; the screenshot UI's Screen and Window modes
  work from the tool bar and keys.
- Windows' screen positions are computed from the layout (the primary
  monitor, the dock's edge) rather than reported by the compositor.
- The tree, roles, names, states, focus and actions are tested over
  AT-SPI on a private accessibility bus, but not yet with Orca on a
  Helix session: whether Orca announces windows and focus changes as
  intended (activation of layer surfaces, live messages, the lock
  screen's password field) is unverified (helix-desktop#71).
