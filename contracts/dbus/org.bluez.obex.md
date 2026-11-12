# org.bluez.obex (Bluetooth file transfer)

BlueZ's OBEX daemon (`obexd`) owns `org.bluez.obex` on the **session**
bus. Helix uses it to send files to a Bluetooth device and to receive
files a device pushes (Object Push Profile, OPP). Helix implements an
`org.bluez.obex.Agent1` and drives the client interfaces; it does not
implement obexd itself.

The interfaces below are described from BlueZ's published API
documentation (`obex-*.rst`), in Helix's own words, listing only what Helix
uses.

## Interfaces Helix calls

### org.bluez.obex.Client1 (`/org/bluez/obex`)

- `CreateSession(s destination, a{sv} args) → o session`:
  `destination` is the device's Bluetooth address (`XX:XX:XX:XX:XX:XX`);
  `args["Target"]` is `"opp"` for Object Push. The returned object has
  `Session1` and, for OPP, `ObjectPush1`.
- `RemoveSession(o session)`: ends the session.

### org.bluez.obex.ObjectPush1 (on a session)

- `SendFile(s sourcefile) → (o transfer, a{sv} properties)`: queues the
  file at absolute path `sourcefile`; the reply's properties are the
  transfer's (as below).

### org.bluez.obex.Session1

- `Destination` (`s`): the remote device's address.

### org.bluez.obex.Transfer1

| Property | Type | Meaning |
|----------|------|---------|
| `Status` | `s` | `queued`, `active`, `suspended`, `complete` or `error` |
| `Name` | `s` | the object's name (the file name the sender gave) |
| `Size` | `t` | size in bytes (0 when unknown) |
| `Transferred` | `t` | bytes so far |
| `Filename` | `s` | the local file (once known) |
| `Session` | `o` | the transfer's session |

`Status` changes are announced with `PropertiesChanged`; a transfer object
goes away soon after it ends. `Cancel()` stops a transfer.

### org.bluez.obex.AgentManager1 (`/org/bluez/obex`)

- `RegisterAgent(o agent)`, `UnregisterAgent(o agent)`: one agent decides
  on incoming pushes.

## Interface Helix implements: org.bluez.obex.Agent1

Exported by the shell at `/org/helix/ObexAgent` on the session bus.

- `AuthorizePush(o transfer) → s`: a device wants to push the transfer's
  object. Returning a name accepts it (a bare name is stored in obexd's
  own folder); failing with `org.bluez.obex.Error.Rejected` or
  `org.bluez.obex.Error.Canceled` refuses it.
- `Cancel()`: obexd gave up on the pending `AuthorizePush` (the device
  went away or the request timed out).
- `Release()`: obexd no longer uses the agent.

## Helix

### Receiving

- The shell registers its agent with `AgentManager1` whenever
  `org.bluez.obex` appears on the session bus. In a session on real
  hardware (the compositor's DRM backend, `HELIX_BACKEND=drm`) the shell
  also starts obexd through D-Bus activation once the session begins; in
  a nested session it only uses an obexd that is already running.
  `HELIX_NO_OBEX` in the environment turns receiving off.
- Every push is asked about: a dialog "Receive a File?" names the device
  (BlueZ's alias for the session's `Destination`, else the address), the
  file name and its size, with **Accept** and **Decline**. Declining, or
  obexd's `Cancel`, refuses the push.
- An accepted file is first stored by obexd under a safe bare name (no
  directories or leading dots). When the transfer completes, the shell
  moves it into the user's Downloads folder (`XDG_DOWNLOAD_DIR`, else
  `~/Downloads`), adding ` (2)`, ` (3)`… before the extension if the name
  is taken, and posts a notification "File received" naming the file and
  the device. A failed transfer posts "File not received" and removes
  the partial file.

### Sending

- The shell serves `org.helix.BluetoothSendTo` at
  `/org/helix/BluetoothSendTo` on its session bus name `org.gnome.Shell`:
  - `SendFiles(s device, as files)`: `device` is a Bluetooth address or
    empty; `files` are absolute paths or `file://` URIs, or empty. With
    no device, the shell asks which paired device to send to (BlueZ's
    paired devices, by alias); with no files, it asks for files with
    Helix's file chooser (the FileChooser portal backend). The call
    returns at once; the rest happens in the shell. It fails with
    `org.freedesktop.DBus.Error.Failed` when sending is not available.
- The files go one after another in one OPP session. A notification
  "Sending files to <device>" shows which file is being sent and how far
  it has got (updated at most every 10%); when all are sent it becomes
  "Files sent to <device>", or "Sending failed" with obexd's error.
- `helix-bluetooth-sendto [--device=ADDRESS] [--name=NAME] [FILE…]` is a
  command-line front end with the same arguments as the usual
  `bluetooth-sendto`, so file managers' "Send to Bluetooth device" works;
  it calls `SendFiles` (`--name` is accepted and ignored).
- The quick settings Bluetooth list has a **Send Files…** row while any
  device is paired; it calls `SendFiles("", [])`.

### Testing

Sending and receiving are tested against a fake `org.bluez.obex` on a
private bus, and the dialogs and notifications in a nested session with
fake obexd and BlueZ services. **Not yet tested with a real Bluetooth
device** (`gaps.md`).
