# org.freedesktop.NetworkManager.SecretAgent — behavioral notes

NetworkManager asks the user's secret agent for passwords it does not
have (a new Wi-Fi network, a changed password). In GNOME the shell is an
agent; in Helix the shell is too, for Wi-Fi. Shapes in
`org.freedesktop.NetworkManager.SecretAgent.xml`; constants as libnm
publishes them.

## Registering

The shell serves the interface on the system bus at
`/org/freedesktop/NetworkManager/SecretAgent`, then calls
`RegisterWithCapabilities("org.gnome.Shell.NetworkAgent", 0)` on
`/org/freedesktop/NetworkManager/AgentManager`
(`org.freedesktop.NetworkManager.AgentManager`). When NetworkManager
starts again (its name gets a new owner) the shell registers again.
NetworkManager accepts one agent per identifier and user.

## GetSecrets

`GetSecrets(connection, connection_path, setting_name, hints, flags) ->
secrets`: `connection` is the connection's settings by group
(`connection`, `802-11-wireless`, `802-11-wireless-security`, …);
`setting_name` names the group whose secrets are wanted. `flags`: 0x1
interaction allowed, 0x2 the stored secret was wrong (ask anew), 0x4 the
user asked for this connection, 0x8 WPS push button active, 0x40000000
no errors, 0x80000000 system secrets only. Errors:
`org.freedesktop.NetworkManager.SecretAgent.UserCanceled`, `.NoSecrets`.

Helix answers, for `802-11-wireless-security`, when interaction is
allowed:

- `key-mgmt` `wpa-psk` or `sae`: asks for `psk`; `none` (WEP): asks for
  `wep-key0`. Other key managements (802.1X enterprise, OWE) and other
  setting groups: `NoSecrets`.
- The dialog: title "Wi-Fi Network", message "A password is needed to
  connect to “<SSID>”." (the SSID from `802-11-wireless` `ssid`), a
  "Password" field, buttons "Connect" and "Cancel"; with flag 0x2 the
  warning "Wrong password? Try again.".
- A WPA password must be 8 to 63 characters, or 64 hexadecimal digits; a
  WEP key 5 or 13 characters, or 10 or 26 hexadecimal digits. Others are
  refused in the dialog with the warning "This password is not valid."
- The answer: `{"802-11-wireless-security": {"psk": <s>}}` (or
  `wep-key0`); Cancel answers `UserCanceled`. Without interaction:
  `NoSecrets`.
- `CancelGetSecrets` closes the dialog that is open for that connection
  and setting (the pending call answers `AgentCanceled`).
- `SaveSecrets` and `DeleteSecrets` succeed without storing anything:
  Helix keeps no secrets itself (NetworkManager keeps system-owned ones).
