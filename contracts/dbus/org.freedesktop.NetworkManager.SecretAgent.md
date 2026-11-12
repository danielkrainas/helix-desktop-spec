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
- `gsm` and `cdma` (mobile broadband): asks for `password`; when
  `hints` holds `pin`, for the SIM card's `pin` (in `gsm`) instead.
- `pppoe` (DSL): asks for `username` (filled with the connection's) and
  `password`.
- `vpn`: see "VPN secrets" below.
- `wireguard`: the keys `hints` name (`private-key`;
  `peers.<public key>.preshared-key`), else the private key; each in its
  own dialog ("WireGuard Connection", "The private key is needed to
  connect to “<id>”." or "The preshared key for peer <first 8 characters
  of its public key>… is needed to connect to “<id>”.", a "Private key" or
  "Preshared key" field). A key must be 32 bytes in base64 (43 characters
  and `=`), else "This key is not valid.". The answer is
  `{"wireguard": {"private-key": <s>, "peers": [{"public-key": <s>,
  "preshared-key": <s>}, …]}}` (only what was asked).
- Other key managements (OWE) and other setting groups (e.g. `adsl`,
  `bluetooth`): `NoSecrets`.
- The dialog: title "Wi-Fi Network" (or "Wired Network" for a connection
  without an `802-11-wireless`, `gsm` or `cdma` group; "Mobile Broadband"
  for mobile broadband; "DSL Connection" for PPPoE); message "A password is needed to
  connect to “<name>”.", or "A user name and password are needed to
  connect to “<name>”.", or for TLS "Your identity and the password of
  your private key are needed to connect to “<name>”.", or for a PIN
  "The SIM card's PIN is needed to connect to “<name>”." (the name: the
  SSID from `802-11-wireless` `ssid`, else the connection's `id`); a
  "User name" text field (filled with the connection's `identity`,
  `leap-username` or PPPoE `username`; not for WPA, WEP and mobile
  broadband) and a "Password" field ("Private key password" for TLS,
  "PIN" for a PIN), buttons "Connect" and "Cancel"; with flag 0x2
  the warning "Wrong password? Try again.". Focus starts in the user name
  field when it is empty, else the password.
- A WPA password must be 8 to 63 characters, or 64 hexadecimal digits; a
  WEP key 5 or 13 characters, or 10 or 26 hexadecimal digits; a PIN 4
  to 8 digits ("A PIN has 4 to 8 digits."); other
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

## VPN secrets

NetworkManager asks for a VPN connection's secrets with `setting_name`
`vpn`. Only the VPN's plugin knows which secrets it needs, so agents run
the plugin's *auth dialog*. The protocol below was observed from the
OpenVPN plugin's auth dialog (network-manager-openvpn 1.10).

- **Finding the dialog.** Each plugin installs a key file `<name>.name`
  in `/usr/lib/NetworkManager/VPN/` (or `/etc/NetworkManager/VPN/`):
  group `[VPN Connection]` with `name` and `service` (the D-Bus service
  that `vpn` `service-type` names), and group `[GNOME]` with
  `auth-dialog` (the program), `supports-external-ui-mode` and
  `supports-hints` (booleans).
- **Running it.** Arguments: `-u <connection uuid> -n <connection id> -s
  <service>`, then `-i` when interaction is allowed, `-r` when the stored
  secrets were wrong (flag 0x2), `--external-ui-mode` when supported (and
  interaction is allowed), and `-t <hint>` for each of `hints` when hints
  are supported. On standard input it gets the `vpn` group's `data`
  entries, each as `DATA_KEY=<key>` and `DATA_VAL=<value>` lines followed
  by an empty line, then the `secrets` entries as `SECRET_KEY=` /
  `SECRET_VAL=` likewise, then `DONE` and an empty line.
- **External UI mode.** The dialog prints a key file and exits: group
  `[VPN Plugin UI]` with `Version` (2), `Title` and `Description`, then
  one group per secret, named by the secret's key, with `Value` (its
  current value), `Label`, `IsSecret` and `ShouldAsk`. The agent asks for
  the secrets whose `ShouldAsk` is true. When none needs asking the
  dialog lists a placeholder (OpenVPN: `[no-secret]` with `Value=true`).
- **Its own dialog.** Without external UI mode the auth dialog asks the
  user itself (or, without `-i`, looks up saved secrets) and prints each
  secret as a key line and a value line, ending with an empty line; it
  then waits for `QUIT` and an empty line on standard input.
- **The answer** is `{"vpn": {"secrets": <a{ss}>}}`.

### Helix

- The plugin is the first `.name` file (in name order) whose `service`
  equals `service-type`, or whose `name` equals it or its last
  dot-separated part; `/etc/NetworkManager/VPN/` is searched first, then
  `/usr/lib/NetworkManager/VPN/` and `/usr/lib64/NetworkManager/VPN/`.
  Without a plugin, an `auth-dialog`, or one that can be run:
  `NoSecrets`.
- External UI mode: the fields to ask are shown in the shell's prompt
  dialog (title and message from the key file, else "VPN Connection" and
  "A password is needed to connect to “<id>”."), each field labelled by
  its `Label` (else its key) and filled with its `Value`. A dialog holds
  at most one text field (`IsSecret` false) and two secret fields, in the
  key file's order; more fields take further dialogs, one after another.
  Buttons "Connect" and "Cancel" (`UserCanceled`). The answer holds every
  asked field and every other field that has a value. The dialog must
  describe itself within 20 seconds, or `NoSecrets`.
- Without external UI mode the plugin's own dialog shows (on the
  session's display), and its secrets are the answer; none (the user
  cancelled it) answers `UserCanceled`, or `NoSecrets` without
  interaction. The auth dialog then gets `QUIT`.
- `CancelGetSecrets` for the connection's `vpn` setting stops the auth
  dialog and closes Helix's dialog (`AgentCanceled`).
- Helix does not keep VPN secrets: those the user wants saved are for the
  plugin's dialog or NetworkManager to keep.
