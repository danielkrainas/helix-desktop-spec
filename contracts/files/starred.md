# Starred files

Helix keeps starred files where GNOME Files (Nautilus) keeps them, so a
file starred in either is starred in both. Helix Files stars and unstars
files and lists them in its Starred place. The file chooser (the shell's
dialog behind the FileChooser portal, `contracts/dbus/org.freedesktop.impl.portal.md`,
"FileChooser") lists them in a Starred place too, read-only.

*GNOME does not document this; Helix's choice:* GNOME publishes no
contract for stars. They live in a private TinySPARQL (Tracker 3)
database that only Nautilus uses and that it does not offer on D-Bus.
Helix reads and writes that database directly, as Nautilus does. It
follows Nautilus 50.2.2's tag manager (`src/nautilus-tag-manager.c`) and
its ontology (`data/ontology/nautilus.ontology`, installed in
`/usr/share/nautilus/ontology`). Without this, stars would not be shared
with GNOME Files. A store of Helix's own would split them in two. The
code is `stars.rs`, one identical file in helixdesktop/desktop
(`crates/helix-settings`) and helixdesktop/apps (`apps/files`).

## The store

- Location: `$XDG_DATA_HOME/nautilus/tags` (`g_get_user_data_dir()`),
  holding `meta.db` with its `-wal` and `-shm` files. It is a TinySPARQL
  direct (local) database in SQLite's WAL mode.
- Nautilus opens it with `tracker_sparql_connection_new` and no flags,
  passing its ontology folder. TinySPARQL is `libtinysparql-3.0`
  (`libtracker-sparql-3.0` before 3.8). Helix loads it at run time, so
  without the library there are no stars and Helix still starts.
- Ontology: namespace `https://gitlab.gnome.org/GNOME/nautilus#`, prefix
  `nautilus`, `nrl:lastModified "2020-05-01T10:00:00Z"`. It defines:
  - the class `nautilus:File`, a subclass of `rdfs:Resource` with
    `nrl:notify true`;
  - the property `nautilus:starred`, with domain `nautilus:File`, range
    `xsd:boolean` and `nrl:maxCardinality 1`.
- A starred file is the resource named by its GIO URI (`g_file_get_uri`,
  that is `g_filename_to_uri` for a local file) with
  `<uri> a nautilus:File ; nautilus:starred true`. An unstarred file has
  no such triples.
- Queries, as Nautilus writes them:
  - stars: `SELECT ?file { ?file a nautilus:File ; nautilus:starred true . }`
  - star: `INSERT DATA { <uri> a nautilus:File ; nautilus:starred true . … }`
  - unstar: `DELETE DATA { <uri> a nautilus:File ; nautilus:starred true . … }`
  - moved or renamed: one update, `DELETE DATA {…} ; INSERT DATA {…}`,
    for the file and every starred file inside it (its URI followed by
    `/`).
- Only files inside the home folder are starred and listed (Nautilus's
  `can_star_location`), not the home folder itself. Stars left on files
  that have gone are kept but not listed.

## Opening without changing Nautilus's database

Helix never migrates the database or changes its shape. TinySPARQL
changes a database when it is opened for writing with an ontology that
differs from the stored one, and also when its internal format is older,
when the collation locale changed (it rebuilds the indexes), or when the
full-text flags changed. So Helix opens it like this:

1. **Read-only first, with no ontology**
   (`TRACKER_SPARQL_CONNECTION_FLAGS_READONLY`): TinySPARQL takes the
   schema from the database and refuses one whose internal format it
   would have to update. Any failure turns stars **off**.
2. **Schema check**, by SPARQL against the stored schema:
   `nautilus:File` must be a class, and `nautilus:starred` a property of
   it, a boolean with at most one value. Anything else (missing, another
   domain, range or cardinality) turns stars **off**: none are shown and
   nothing is written.
3. **Writing** (Helix Files, and the shell for Agent Access's
   `StarFile`) needs both of these; otherwise stars
   are **read-only**:
   - the stored schema version (`nautilus: nrl:lastModified`) is the one
     above (a newer Nautilus may mean more than this check sees);
   - opening for writing would rebuild nothing: the database's `locale`
     metadata is this process's `LC_COLLATE`, and its `fts-flags` are
     `0` or absent, as Nautilus leaves them. Helix reads this with SQLite,
     read-only.

   It then opens a writing connection, still with no ontology.
4. **A missing store** means no stars. It is created only when Helix
   Files stars something, as Nautilus creates it (no flags, with an
   ontology): with Nautilus's installed ontology folder when there is one
   (only if it is the version above), else with the same schema written
   out by Helix (the same terms; comments of its own). Nautilus opens
   such a store with its own ontology and changes nothing but the
   comments. The shell creates it the same way for Agent Access's
   `StarFile`; the file chooser never creates it.

The chooser opens the store read-only for each listing. It shows the
Starred place (after Recent, when Recent is offered) while `meta.db`
exists. A listing of a store that is off answers an error.

## Concurrency with a running Nautilus

Checked with Nautilus 50.2.2 and TinySPARQL 3.11 against a scratch copy,
never a user's store. A real Nautilus process (`--gapplication-service`
on a private bus and a broadway display) held the store open. Nautilus's
own renames (`org.gnome.Nautilus.FileOperations2.RenameURI`) wrote to it,
as did a writer driving the library exactly as Nautilus does and Helix's
module, with Helix readers alongside:

- **Both sides can read and write at once.** In WAL mode, readers never
  block and every write is visible to the others' next read. 3000 stars
  and 1000 unstars on each side, while 300 read-only opens ran alongside,
  ended with exactly the expected 4003 stars.
- **Writers can collide.** TinySPARQL begins transactions deferred, so
  when two writers meet SQLite answers "database is locked" at once
  instead of waiting (TinySPARQL's 100 s busy timeout does not apply).
  Helix retries such writes, up to 20 times with growing pauses. Nautilus
  does not retry, so a star made in Nautilus at the same instant is lost
  with a warning. It happened 8 times in the stress run above, and is
  rare at human speed.
- **Helix sees Nautilus's changes.** Helix Files looks at the size and
  time of `meta.db` and `meta.db-wal` every 2 s and reads the stars again
  when they change. Every write appends to the WAL first.
- **Nautilus does not see Helix's changes while it runs.** It loads its
  stars at startup and then follows only its own connection's
  `TrackerNotifier`. So it shows Helix's stars only after it restarts,
  and its renames move only the stars it knows of
  (helixdesktop/desktop#154).
- **The locale matters.** A writer whose collation locale differs from
  the stored one makes TinySPARQL rebuild the indexes on open. The test
  run saw this between processes started with different locales. Helix
  turns read-only instead (step 3). Apps of one session normally share
  the locale.
- TinySPARQL keeps `nrl:modified` sequence numbers per connection, so
  two writers can store the same number. Neither Nautilus nor Helix reads
  them.

## Helix Files

- Starred place in the sidebar, after Recent (`starred:///` inside
  Files). It lists the starred files inside the home folder, with "No
  Starred Files" when there are none and "Starred Files Unavailable" when
  stars are off. Its menus offer Open Item Location and Unstar. Nothing
  can be pasted or dropped there.
- Star or Unstar in a file's context menu: Star when some of the
  selected files are not starred, Unstar when all of them are, and
  neither for files outside the home folder or when stars are read-only.
  Properties has a star toggle under the name (named "Starred", pressed
  while starred). Each change is announced ("Starred", "Unstarred").
- A star after the name in the list view and in the icon's corner in the
  grid, named "Starred" for assistive technology.
- Renames, moves and undoing a rename move the stars, as Nautilus's
  `update_moved_uris` does. Trashing does not unstar (nor does Nautilus),
  so a restored file is still starred.
- Not yet: undo for stars, a clickable star in the list, and search in
  Starred (helixdesktop/desktop#155).

## Agent Access

The shell reads the stars for agents (`StarredFiles`, opening the store
read-only as the file chooser does) and, from batch 2, stars and unstars
files for them (`StarFile`), writing as Helix Files does: under the same
conditions (step 3), and creating a missing store as in step 4. See
`contracts/agent-access/README.md`, "Media, power and stars" and
"Changing settings".
