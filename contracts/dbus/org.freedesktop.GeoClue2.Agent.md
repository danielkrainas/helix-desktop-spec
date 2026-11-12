# org.freedesktop.GeoClue2.Agent — behavioral notes

GeoClue (`org.freedesktop.GeoClue2`, system bus) gives apps their
location only after the user's agent agrees. In GNOME the shell is that
agent; in Helix the shell is too. Interface shape from GeoClue's published
D-Bus interface files (`org.freedesktop.GeoClue2.Agent.xml`,
`org.freedesktop.GeoClue2.Manager.xml`).

## Accuracy levels

GeoClue's levels, as `u`: 0 none, 1 country, 4 city, 5 neighborhood,
6 street, 8 exact. `org.gnome.system.location max-accuracy-level` names
them (`country` … `exact`).

## Registering

- The agent serves `org.freedesktop.GeoClue2.Agent` on the system bus at
  `/org/freedesktop/GeoClue2/Agent`, then calls the Manager's
  `AddAgent(id)` (`/org/freedesktop/GeoClue2/Manager`). GeoClue accepts
  only ids its configuration lists (`[agent] whitelist` in
  `/etc/geoclue/geoclue.conf`); Helix registers as `gnome-shell`, the id
  configurations list for GNOME (see `gaps.md`).
- One agent per user. When GeoClue starts again (its name gets a new
  owner) the agent registers again.

## The interface

- Property `MaxAccuracyLevel` (`u`, read): the most any app may get. Helix:
  0 while `org.gnome.system.location enabled` is false, otherwise the level
  `max-accuracy-level` names. Changes are signalled with
  `org.freedesktop.DBus.Properties.PropertiesChanged`.
- `AuthorizeApp(desktop_id s, req_accuracy_level u) -> (authorized b,
  allowed_accuracy_level u)`: may the app (desktop file id without
  `.desktop`) have its location, and how exactly. GeoClue asks only for
  apps it does not treat as system components; it waits for the answer.

## Helix's answers

- Location services off (`enabled` false): not authorized, level 0.
- An id with no installed desktop file: not authorized, without asking.
- Otherwise the user's earlier decision for the app, kept in the
  permission store (`org.freedesktop.impl.portal.PermissionStore`, table
  `location`, id `location`, the app's desktop id): a first value of
  `NONE` refuses; one of `COUNTRY`, `CITY`, `NEIGHBORHOOD`, `STREET`,
  `EXACT` allows up to that level.
- With no decision, a dialog asks: title "Location Access", message
  "<app name> wants to use your location.", description "Location access
  can be changed at any time from the privacy settings.", buttons "Allow"
  and "Deny". Allowing stores the requested level (capped at the maximum),
  denying stores `NONE`. The dialog takes its turn with other prompts.
- When allowed, the level returned is the smallest of the requested
  level, the stored one and `MaxAccuracyLevel`.

## Location in use

While location services are on and the Manager's `InUse` (`b`) is true,
the panel's status area shows a location icon (`location-services-active-symbolic`).
The shell does not start GeoClue to find out: it reads `InUse` only while
`org.freedesktop.GeoClue2` has an owner.
