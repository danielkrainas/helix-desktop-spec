# org.gnome.Shell.SearchProvider2 — behavioral notes

Apps serve this interface so the shell's search shows their results (files,
settings panels, calculations, characters…). The shell is the client.

## Finding providers

An app installs a key file in `gnome-shell/search-providers/` under an XDG
data directory, group `[Shell Search Provider]`:

- `DesktopId`: the app's desktop file id;
- `BusName`, `ObjectPath`: where the provider is (the bus may start it);
- `Version`: 2 for this interface;
- `DefaultDisabled` (optional, `true`/`false`): off unless enabled.

`org.gnome.desktop.search-providers`
(`contracts/gsettings/org.gnome.desktop.search-providers.md`):
`disable-external` turns all providers off; `disabled` lists desktop ids
turned off; `enabled` lists `DefaultDisabled` providers turned on;
`sort-order` lists desktop ids to show first, in that order.

## Methods

- `GetInitialResultSet(terms)`: result ids for the search terms.
- `GetSubsearchResultSet(previous_results, terms)`: the same for terms
  that refine the previous ones (narrowing `previous_results`).
- `GetResultMetas(identifiers)`: one dictionary per id: `id` (`s`),
  `name` (`s`), `description` (`s`, optional), and the icon as `icon` (a
  serialized `GIcon`, `v`), `gicon` (`s`, a `GIcon` string) or
  `icon-data` (`(iiibiiay)`); also `clipboardText` (`s`, optional).
- `ActivateResult(identifier, terms, timestamp)`: the user chose a result.
- `LaunchSearch(terms, timestamp)`: open the app with the search.

## Helix

- Providers come from `$XDG_DATA_HOME` and each of `$XDG_DATA_DIRS` (a file
  name found earlier wins), with `Version=2` and a `DesktopId` of an
  installed app; they are read when the launcher opens (also when typing
  in the overview opens it).
- Order: the `sort-order` ids in that order, then the others by app name.
- The launcher (see `contracts/wayland/README.md`, "Window switching")
  asks every enabled provider at once on each change of its search field
  (not for an empty one). Terms are the field's text split at whitespace.
  When the new terms refine the provider's previous ones (as many or more
  terms, each previous term a prefix of the new one at its place) it uses
  `GetSubsearchResultSet` with the previous results, else
  `GetInitialResultSet`. A provider that has not answered within 5
  seconds, or whose answer is for an older text, is left out.
- Up to 3 results per provider are shown, after the windows and apps,
  grouped by provider in order: each its `name`, its `description` (in a
  dimmer color) and, at the row's end, the provider's app name. Choosing
  one (Enter or a click) calls `ActivateResult` (timestamp 0) and closes
  the launcher.
- Result icons, `clipboardText` and `LaunchSearch` are not used.
