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

Helix answers, when interaction is allowed:

- `802-11-wireless-security` with `key-mgmt` `wpa-psk` or `sae`: asks for
  `psk`; `none` (WEP): asks for `wep-key0`; `none` or `ieee8021x` with
  `auth-alg` `leap`: asks for `leap-username` and `leap-password`.
- `802-1x` (WPA and WPA2/3 Enterprise, dynamic WEP, wired 802.1X): when
  the first of `eap` is `tls`, asks for `identity` and
  `private-key-password`; otherwise (PEAP, TTLS, PWD, FAST, LEAP, MD5)
  for `identity` and `password`. The certificates themselves are not
  asked for.
- Other key managements (OWE), other setting groups (VPN, mobile
  broadband, PPPoE): `NoSecrets`.
- The dialog: title "Wi-Fi Network" (or "Wired Network" for a connection
  without an `802-11-wireless` group); message "A password is needed to
  connect to “<name>”.", or "A user name and password are needed to
  connect to “<name>”.", or for TLS "Your identity and the password of
  your private key are needed to connect to “<name>”." (the name: the
  SSID from `802-11-wireless` `ssid`, else the connection's `id`); a
  "User name" text field (filled with the connection's `identity` or
  `leap-username`; not for WPA and WEP) and a "Password" field ("Private
  key password" for TLS), buttons "Connect" and "Cancel"; with flag 0x2
  the warning "Wrong password? Try again.". Focus starts in the user name
  field when it is empty, else the password.
- A WPA password must be 8 to 63 characters, or 64 hexadecimal digits; a
  WEP key 5 or 13 characters, or 10 or 26 hexadecimal digits; other
  passwords must not be empty, and a user name is needed where one is
  asked. Others are refused in the dialog with the warning "This password
  is not valid." (or "A user name is needed.").
- The answer: the asked setting group with the keys above, e.g.
  `{"802-11-wireless-security": {"psk": <s>}}` or `{"802-1x":
  {"identity": <s>, "password": <s>}}`; Cancel answers `UserCanceled`.
  Without interaction: `NoSecrets`.
- `CancelGetSecrets` closes the dialog that is open for that connection
  and setting (the pending call answers `AgentCanceled`).
- `SaveSecrets` and `DeleteSecrets` succeed without storing anything:
  Helix keeps no secrets itself (NetworkManager keeps system-owned ones).
