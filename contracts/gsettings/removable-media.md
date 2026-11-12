# Removable media (Helix)

What the Helix shell does when drives and media come and go, following
`org.gnome.desktop.media-handling`
([org.gnome.desktop.media-handling.md](org.gnome.desktop.media-handling.md)).
Volumes and mounts are the ones GIO's volume monitor reports (with gvfs:
the udisks2 monitor), so only media meant to be shown to the user count.
Settings are read when a medium arrives; changes apply to the next one.

## Mounting

- When the session starts, every volume that is not mounted, can be
  mounted and asks to be mounted automatically (GIO's
  `should-automount`) is mounted, with `automount` true. Nothing is opened
  and no notification is shown for these.
- A volume that arrives later is mounted the same way. While the screen is
  locked it waits, and is mounted when the session is unlocked.
- Mounting asks no questions: a volume that needs a passphrase or other
  answers fails to mount and is left alone (see `gaps.md`).

## When a medium is mounted

For a mount that arrives during the session, belongs to a volume and is
not shadowed, the shell guesses its content types (GIO, without looking
through the whole medium), then:

1. If any of them is in `autorun-x-content-ignore`: nothing.
2. Else if `autorun-never` is true: nothing.
3. Else if any is in `autorun-x-content-open-folder`, or the shell mounted
   it itself and `automount-open` is true: its root is opened with the
   default handler for folders.
4. Else: a notification titled with the mount's name, body "Removable
   medium connected", with the buttons "Open with Files" (opens its root as
   in 3) and "Eject".

Types in `autorun-x-content-start-app` are treated as in 4: Helix starts
no programs from media.

## Ejecting

"Eject" ejects the mount when it can be ejected, otherwise unmounts it.
When that succeeds a notification says "<name> can be removed"; when it
fails (e.g. files are still open), one says "Cannot eject <name>" with the
reason. The notification for a mount is withdrawn when the mount goes away.
