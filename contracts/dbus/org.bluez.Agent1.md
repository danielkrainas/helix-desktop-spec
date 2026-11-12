# org.bluez.Agent1 — behavioral notes

BlueZ asks the user's pairing agent to show or ask for pairing codes and
to authorize devices. In GNOME the shell's Bluetooth support (and
Settings) does this; in Helix the shell does. Shapes in
`org.bluez.Agent1.xml`, after BlueZ's API documentation.

## Registering

The shell serves `org.bluez.Agent1` on the system bus at
`/org/helix/BluetoothAgent`, then calls `RegisterAgent(path,
"KeyboardDisplay")` and `RequestDefaultAgent(path)` on `/org/bluez`
(`org.bluez.AgentManager1`). When BlueZ starts again (its name gets a new
owner) the shell registers again. `Release()` is acknowledged.

## Requests

Device names are the device's `Alias` (`org.bluez.Device1`), else its
address. Refusing answers the error `org.bluez.Error.Rejected`; a dialog
closed by `Cancel()` answers `org.bluez.Error.Canceled`.

- `RequestConfirmation(device, passkey)`: dialog "Pair with <name>?",
  message "Confirm that <passkey, six digits> is shown on <name>.",
  buttons "Pair" and "Cancel".
- `RequestAuthorization(device)` (pairing without codes) and
  `AuthorizeService(device, uuid)` (a paired but untrusted device wants a
  service): dialog "<name>" with "<name> wants to pair with this
  computer." or "<name> wants to connect to this computer.", buttons
  "Allow" and "Deny".
- `RequestPinCode(device)`: a "PIN" field (1 to 16 characters);
  `RequestPasskey(device)`: a "Passkey" field (a number from 0 to
  999999); both with "Pair" and "Cancel". Invalid entries are refused in
  the dialog ("This code is not valid.").
- `DisplayPinCode(device, pincode)` and `DisplayPasskey(device, passkey,
  entered)`: a notification "Pair with <name>" ("Type <code> on <name>,
  then press Enter."), once per device until it is paired or canceled.
- `Cancel()`: closes the open dialog and withdraws the code notification.
