# org.gnome.Shell.Extensions — behavioral notes

Companion to `org.gnome.Shell.Extensions.xml`. Extension tools (the
`gnome-extensions` command, the Extensions app) use it to list, enable,
disable and remove GNOME Shell extensions.

- Bus name: `org.gnome.Shell`
- Object: `/org/gnome/Shell`

## Extensions

An extension is a directory named by its UUID holding a `metadata.json`
object. Helix takes one when that object's `uuid` is the directory's
name, `name` and `description` are strings and `shell-version` is an
array. Extensions are found in `$XDG_DATA_HOME/gnome-shell/extensions`
(per-user, type 2), then in each `$XDG_DATA_DIRS` entry's
`gnome-shell/extensions` (system, type 1). The first one found for a
UUID counts, so a per-user copy hides a system one.

## Info

`ListExtensions` maps each UUID to its info; `GetExtensionInfo` gives one
(an empty dictionary for an unknown UUID). The info holds the
metadata's members whose values are strings (`s`), numbers (`d`),
booleans (`b`) or string arrays (`as`), such as `version`, `url`,
`settings-schema` and `gettext-domain`. Then these keys, set by Helix:

- `uuid` (`s`), `path` (`s`): the directory.
- `type` (`d`): 1 system, 2 per-user.
- `state` (`d`) and `error` (`s`), see below.
- `enabled` (`b`): `gnome-extensions` reads this key for "Enabled" and
  for `list --enabled` / `--disabled`.
- `hasPrefs` (`b`): the directory has `prefs.js`.
- `hasUpdate` (`b`): always false.
- `canChange` (`b`): false for a per-user extension while user
  extensions are off, else true.

`gnome-extensions` was seen to read `enabled`, `state`, `hasPrefs` and
the metadata's `name`, `description` and `url` (by running it against a
test service). Which other keys the Extensions app reads is not
recorded here.

## Enabled and state

Enabling is kept in `org.gnome.shell`
(`contracts/gsettings/org.gnome.shell.md`). An extension is enabled when
it is in `enabled-extensions` and not in `disabled-extensions`. A per-user
extension is also not enabled while `disable-user-extensions` is true.
A GNOME session of the same user therefore sees the same choices.

**Helix runs no extension** (they are JavaScript for GNOME Shell's own
engine). The states are:

- not enabled: 2 (INACTIVE), `error` empty;
- enabled, but `shell-version` lists no GNOME 51 version (an entry whose
  first dot-separated part is `51`) and
  `disable-extension-version-validation` is false: 4 (OUT_OF_DATE);
- otherwise enabled: 3 (ERROR), `error` "Helix does not run GNOME Shell
  extensions".

`GetExtensionErrors` gives that error when there is one, else nothing.

## Methods

- `EnableExtension(uuid)`: adds the UUID to the end of
  `enabled-extensions` (once) and removes it from
  `disabled-extensions`. `DisableExtension(uuid)` removes it from both.
  Both return false for an unknown UUID or when the keys cannot be
  written; otherwise true, and they emit `ExtensionStateChanged(uuid,
  info)` and `ExtensionStatusChanged(uuid, state, error)`.
- `UninstallExtension(uuid)`: only a per-user extension. Its directory
  is deleted, it is removed from both lists, and the signals are emitted
  with state 99 (UNINSTALLED). Returns false for system or unknown
  extensions, or when the directory cannot be deleted.
- `ReloadExtension(uuid)`: reads the metadata again and emits the
  signals.
- `InstallRemoteExtension`, `LaunchExtensionPrefs` and
  `OpenExtensionPrefs` fail with
  `org.freedesktop.DBus.Error.NotSupported`. Downloading from
  extensions.gnome.org is not done, and preferences are JavaScript too.
  (`gnome-extensions install` unpacks a file into the per-user directory
  itself, so it works.)
- `CheckForUpdates` does nothing.

## Properties

- `ShellVersion` (`s`): `51.0`, as `org.gnome.Shell`'s.
- `UserExtensionsEnabled` (`b`, read/write): the inverse of
  `disable-user-extensions`. Setting it writes that key, and the signals
  are emitted for every per-user extension.
