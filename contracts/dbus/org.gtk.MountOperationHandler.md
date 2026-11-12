# org.gtk.MountOperationHandler — behavioral notes

Companion to `org.gtk.MountOperationHandler.xml`. A desktop shell serves
this so that GTK apps' mount operations (a password for a network share
or an encrypted disk, a question such as "connect anyway?", the apps that
keep a disk busy when unmounting) are asked in the shell's own dialog. GTK
uses it whenever the bus name has an owner; otherwise it shows its own
dialogs.

- Bus name: `org.gtk.MountOperationHandler`
- Object: `/org/gtk/MountOperationHandler`
- Interface: `org.Gtk.MountOperationHandler` (capital G)

The argument types were observed from GTK 3.24 and 4.14's calls (the
interface is not otherwise documented outside GTK and gnome-shell).

## Calls

- `object_id`: names the operation (GTK: one per `GtkMountOperation`); a
  repeated call with the same id is the same question asked again (GTK
  does this for `ShowProcesses` while the disk stays busy).
- `message`: GIO's message, whose first line is the main text and the rest
  details. `icon_name`: an icon for the dialog (GTK: `drive-harddisk`).
- `flags` (`AskPassword`): GIO's `GAskPasswordFlags`: 1 password needed,
  2 user name needed, 4 domain needed, 8 saving the password is supported,
  16 anonymous login is supported, 32 a TCRYPT (VeraCrypt) volume.
  GTK only calls `AskPassword` when neither a user name nor a domain is
  needed (it asks those in its own dialog).
- `choices` (`AskQuestion`, `ShowProcesses`): the answers to offer.
  `application_pids`: the processes using the disk.
- Each call returns when the user answers: `response` is GIO's
  `GMountOperationResult`: 0 handled (answered), 1 aborted (the user
  cancelled), 2 unhandled. GTK ignores an answer of 2 (it waits for
  another answer, as after a repeated call). `response_details`:
  - `AskPassword`: `password` (`s`), `password_save` (`u`, GIO's
    `GPasswordSave`: 0 never, 1 for the session, 2 permanently), which
    GTK was seen to read. Which keys GTK reads for TCRYPT volumes is not
    recorded here.
  - `AskQuestion`, `ShowProcesses`: `choice` (`i`), the index in `choices`.
- `Close()`: the app no longer needs the dialog (GTK calls it after each
  answer, and when the operation ends without one, e.g. the disk stopped
  being busy). It names no operation.

## Helix

The shell owns the name (once the object is exported) and asks with its
prompt dialog, the one polkit, keyring and removable-media questions use
(dialogs take turns). The title is the message's first line, the text
below it the rest; `icon_name` is not shown.

- **Passwords.** Fields as `flags` ask: with 2 and 4 a "User name
  (domain\name)" field holding `<default_domain>\<default_user>` (or the
  user alone without a default domain); with only 2 "User name"
  (`default_user`); with only 4 "Domain" (`default_domain`); with 1 a
  "Password" field; with 8 a "Remember Password" check box. Buttons
  "Cancel" (answer 1) and "Connect" (answer 0). A request needing none
  of user name, domain or password answers 2. Results: `password` (with 1),
  `password_save` 2 when "Remember Password" was checked, else 0, and
  (for other callers than GTK) `username` and `domain` (`s`): the text up
  to the first `\` is the domain, the rest the user name; without `\` the
  domain is `default_domain`. Anonymous login (16) is not offered;
  TCRYPT volumes (32) get the password only (no hidden-volume, system
  volume or PIM fields).
- **Questions.** With one or two choices they are the dialog's buttons:
  the first continues (answer 0, `choice` 0), the second is the cancel
  button and answers its index (Escape too); with one choice the cancel
  button is "Cancel" and answers 1. With more choices they are rows to
  pick from (the first picked at first; clicking the picked row again
  chooses it), with buttons "Cancel" (answer 1) and "Choose".
- **Busy disk** (`ShowProcesses`): as a question, with "In use by:" and
  the processes' names (`/proc/<pid>/comm`, at most five, then "and N
  more") below the text.
- **Repeated calls.** A call with the same `object_id` from the same
  caller while its dialog is shown replaces the dialog's contents; the
  earlier call answers 2.
- **`Close`** dismisses the dialog shown for the caller (the same bus
  connection), whose call answers 2; it does nothing to another app's
  dialog. A dialog whose app left the bus stays until answered.
