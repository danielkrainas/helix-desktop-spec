# org.freedesktop.PolicyKit1.AuthenticationAgent — behavioral notes

The polkit authentication agent: when an app asks polkitd for an action
that needs the user to authenticate, polkitd asks the agent registered for
the app's session; the agent asks the user for a password. In GNOME the
shell is the agent.

## Registering

- polkitd's `org.freedesktop.PolicyKit1.Authority` (system bus,
  `/org/freedesktop/PolicyKit1/Authority`): `RegisterAuthenticationAgent(
  subject, locale, object_path)`, with `subject` `("unix-session",
  {"session-id": <s>})`, and `UnregisterAuthenticationAgent(subject,
  object_path)`. Only one agent per session; registering fails when one
  exists.
- The agent serves this interface at `object_path` on the system bus.

## Methods

- `BeginAuthentication(action_id, message, icon_name, details, cookie,
  identities)`: authenticate as one of `identities` (`("unix-user",
  {"uid": <u>})`, `("unix-group", {"gid": <u>})`) for `action_id`;
  `message` is for the user. It returns once polkitd has been told the
  result, or with an error when the user cancels
  (`org.freedesktop.PolicyKit1.Error.Cancelled`).
- `CancelAuthentication(cookie)`: polkitd no longer wants that
  authentication; the agent stops asking.
- Proving the password goes through polkit's setuid helper
  (`polkit-agent-helper-1`, in `/usr/lib/polkit-1` or `/usr/libexec`),
  which tells polkitd itself: run it with the user name as its argument,
  write the cookie and a newline to it; it then writes lines
  `PAM_PROMPT_ECHO_OFF <prompt>` or `PAM_PROMPT_ECHO_ON <prompt>` (answer
  with a line), `PAM_ERROR_MSG <text>`, `PAM_TEXT_INFO <text>`, and finally
  `SUCCESS` or `FAILURE`.

## Helix

- The shell registers for its own logind session (from logind
  `GetSessionByPID`) at `/org/helix/PolicyKit1/AuthenticationAgent`,
  with the locale of `LC_ALL`, else `LC_MESSAGES`, else `LANG`, else
  `C`, when it starts; polkitd forgets the agent when the shell's system
  bus connection closes. When another agent is
  registered, Helix logs it and goes without. The environment variable
  `HELIX_NO_POLKIT_AGENT` (any value) keeps it from registering (for
  nested test sessions).
- The identity: the user running the session when it is among the
  `unix-user` identities, else the first `unix-user` one; groups are
  not offered. Without a `unix-user` identity the call fails with
  `org.freedesktop.PolicyKit1.Error.Failed`.
- Only the owner of `org.freedesktop.PolicyKit1` may call the agent;
  others get `org.freedesktop.PolicyKit1.Error.NotAuthorized`.
- One authentication at a time; others wait their turn.
- The helper is the first of the paths above that exists
  (`HELIX_POLKIT_HELPER` names another, for tests).
- The user sees the prompt dialog (below): title "Authentication
  Required", `message`, the user's name (real name, else login name), a
  password field labelled with the helper's prompt (without a trailing
  colon and space), Cancel and Authenticate. `PAM_TEXT_INFO` and
  `PAM_ERROR_MSG` texts show in the dialog. After `FAILURE` the dialog asks
  again with "Sorry, that didn't work. Please try again."; Cancel (or
  Escape) ends with `Cancelled`.

## Prompt dialog (Helix)

The shell's dialog for passwords (also used for keyring prompts): a box in
the middle of the primary monitor over a dimmed screen, with a title, a
message, a smaller description, an optional text field (shown as typed,
e.g. a user name), a password field (bullets instead of characters), an
optional second field to confirm a new password, an optional check box, a
warning line, and Cancel / continue buttons. Enter continues, Escape
cancels, Tab (or a click) moves between fields, Space toggles the check
box when no field has text focus. While the answer is checked the dialog
shows "Authenticating…" and ignores input.
