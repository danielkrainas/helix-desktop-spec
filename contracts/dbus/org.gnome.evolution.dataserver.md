# Evolution Data Server — what Helix calls

Evolution Data Server (EDS) keeps the user's calendars. Helix's shell reads
events from it for the message tray (`contracts/gsettings/calendar.md`).
EDS's D-Bus interfaces are private to EDS's own client library and change
version with it; Helix uses the ones EDS 3.5x serves (session bus):

## Sources

`org.gnome.evolution.dataserver.Sources5` at
`/org/gnome/evolution/dataserver/SourceManager` is an
`org.freedesktop.DBus.ObjectManager`. Each object has the interface
`org.gnome.evolution.dataserver.Source` with properties `UID` (`s`) and
`Data` (`s`, a key file). A calendar's key file has a `[Calendar]` group;
it is used when `[Data Source] Enabled` and `[Calendar] Selected` are not
`false`.

## Calendars

- `org.gnome.evolution.dataserver.Calendar8` at
  `/org/gnome/evolution/dataserver/CalendarFactory`, interface
  `org.gnome.evolution.dataserver.CalendarFactory`:
  `OpenCalendar(source_uid s) -> (object_path s, bus_name s)`.
- The calendar there, interface `org.gnome.evolution.dataserver.Calendar`:
  `Open() -> (properties as)` first, then `GetObjectList(query s) ->
  (ics_objects as)`. Helix's query is
  `(occur-in-time-range? (make-time "<from>") (make-time "<to>"))`, times
  as `YYYYMMDDTHHMMSSZ`. The answer is iCalendar text (RFC 5545): events,
  recurring ones as their master (and detached instances), not expanded.
- `TZID`s are Olson names, possibly behind a prefix such as
  `/freeassociation.sourceforge.net/`.
