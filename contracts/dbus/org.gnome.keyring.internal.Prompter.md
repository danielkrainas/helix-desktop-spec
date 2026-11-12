# org.gnome.keyring.internal.Prompter — behavioral notes

gcr's system prompter: how gnome-keyring (and other gcr users) ask the
desktop for a password, e.g. to unlock the login keyring or for an SSH key's
passphrase. In GNOME the shell serves it; without one, D-Bus activates
gcr's own GTK prompter.

## Names and objects

- Bus name (session bus): `org.gnome.keyring.SystemPrompter`.
- The prompter: `/org/gnome/keyring/Prompter`, interface
  `org.gnome.keyring.internal.Prompter`.
- Each client serves `org.gnome.keyring.internal.Prompter.Callback` at the
  object path it passes as `callback`.

## Flow

1. The client calls `BeginPrompting(callback)`; it returns at once.
2. When the prompter is ready for that client, it calls the callback's
   `PromptReady("", {}, exchange)`, `exchange` holding the prompter's
   public key.
3. The client calls `PerformPrompt(callback, type, properties, exchange)`
   (`type` `password` or `confirm`; `properties` the prompt properties
   that changed; `exchange` holding the client's public key); it returns at
   once. The prompter shows the prompt.
4. When the user answers, the prompter calls `PromptReady(reply,
   properties, exchange)`: `reply` `yes` (continue) or `no` (cancel);
   `properties` the properties the prompt changed (`choice-chosen`,
   `password-strength`; each value wrapped in a second variant, as gcr
   reads them); for a password, `exchange` holds the password
   encrypted.
5. The client may perform more prompts (e.g. after a wrong password, with
   a `warning`), then calls `StopPrompting(callback)`; the prompter closes
   the prompt and calls `PromptDone()`.

Errors: `org.gnome.keyring.Prompter.InProgress` (prompting already
active for another client), `org.gnome.keyring.Prompter.Failed`.

### Prompt properties

`title`, `message`, `description`, `warning` (strings); `password-new`
(`b`: choosing a new password, to be typed twice); `password-strength`
(`i`, set by the prompter); `choice-label` (a check box's label; empty
for none), `choice-chosen` (`b`); `caller-window` (`s`, the window the
prompt is for); `continue-label`, `cancel-label` (button labels).

## Secret exchange (`sx-aes-1`)

The `exchange` strings are GLib key files with one group, `[sx-aes-1]`,
and base64 values:

- `public`: the sender's Diffie-Hellman public key (big-endian, no
  leading zero bytes) in the 1536-bit MODP group of RFC 3526 (generator
  2);
- `secret` and `iv`: the secret, encrypted.

The shared secret (big-endian, zero-padded on the left to 192 bytes) goes
through HKDF-SHA256 (no salt, no info) to a 16-byte AES-128 key; the
secret is encrypted with AES-128-CBC, a random 16-byte `iv`, and PKCS#7
padding.

## Helix

- The shell owns `org.gnome.keyring.SystemPrompter` (taking it over from a
  running gcr prompter).
- One client prompts at a time, and polkit and keyring prompts take turns
  too; others wait (their `PromptReady` comes when it is their turn).
- The prompt dialog (`org.freedesktop.PolicyKit1.AuthenticationAgent.md`,
  "Prompt dialog") shows `title` (else "Unlock"), `message`,
  `description`, `warning`, the check box when `choice-label` is set, and
  the button labels (else "Continue" and "Cancel"). A `password` prompt
  has a password field ("Password"), and with `password-new` a second one
  ("Confirm"); entries that differ are asked again with the warning
  "The passwords do not match." A `confirm` prompt has no field.
- `password-strength` is 0 for an empty password, else 1.
- When a client leaves the bus while prompting, its prompt closes and the
  next client goes on.
