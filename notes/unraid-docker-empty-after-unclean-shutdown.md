# Unraid: Docker shows enabled with zero containers after an unclean shutdown

## Symptom

Server lost power (UPS couldn't hold it through the outage). On restart:
the array came up, parity check started automatically as expected, but
Docker showed as **enabled** in Settings while the Docker tab listed **zero
containers**. No obvious error on the Docker settings page itself.

## The error to look for

Tools > System Log (or `/var/log/syslog`) was flooded, once a second, with:

```
emhttpd: error: malloc_share_locations, 8948: Operation not supported (95): getxattr: /mnt/user/<share>
```

In this case it was hitting exactly two shares repeatedly: `appdata` and
`system`, the two shares that back Docker's application data and the
docker image/directory. A third, unrelated share was also affected. Every
other share on the box was unaffected.

## Root cause

One physical drive had come back from the power event in a broken state:
an enterprise SAS drive formatted with a **520-byte native sector size**
instead of the standard 512-byte sectors Linux expects (dmesg showed
`Unsupported sector size 520` / `Invalid physical block size (520)`). The
kernel reported it as 0 B capacity. It sat in Unraid's Unassigned Devices
list, not a member of any pool or the array.

That shouldn't have mattered; it wasn't part of any share's storage. But
`emhttpd`'s periodic share-location scan (`malloc_share_locations`) walks
*all* currently visible block devices, including unassigned ones, while
resolving where each user share's data actually lives. Hitting the dead
drive's zero-size/unsupported geometry threw `getxattr: Operation not
supported` on every pass. Because two of the affected shares were the ones
backing Docker, Docker's daemon started cleanly against storage it
couldn't actually resolve, logged that containers had started, and had
nothing to start.

The array and all real storage pools stayed green and fully healthy the
entire time. This was purely a share-resolution problem, not data loss or
pool damage.

## What didn't work: just pulling the drive

Physically removing the dead drive did **not** clear the errors. They
continued at the same rate immediately after removal (confirmed by pulling
diagnostics before and after and comparing live log timestamps).

The reason: `emhttpd` builds its device table once at array start and
keeps it in memory for the life of the process. Hot-removing a device
doesn't make an already-running instance notice or refresh that table.

## The actual fix

A full **Stop Array → Start Array** cycle is required to force `emhttpd`
to rebuild its device and share state from current hardware. This is
non-destructive and safe: it unmounts and remounts cleanly, and if a
parity check is in progress, Unraid offers to resume it rather than
starting over.

## If Stop Array hangs

Stopping the array can itself get stuck, looping this every few seconds:

```
emhttpd: shcmd: umount /mnt/user
root: umount: /mnt/user: not mounted.
emhttpd: shcmd: rmdir /mnt/user
root: rmdir: failed to remove '/mnt/user': Directory not empty
emhttpd: Retry unmounting user share(s)...
```

Before assuming a process has a file open under `/mnt/user`, check:

```
ps aux | grep -iE "dockerd|docker-containe|shfs|smbd|nfsd"
lsof /mnt/user 2>/dev/null
```

If both come back empty, nothing is actually holding it open. `/mnt/user`
itself is normally just the mountpoint `shfs` overlays onto real storage.
During the broken window, before `shfs` could properly attach, some
processes wrote directly onto the bare mountpoint instead of being routed
through to real disk, leaving a few KB of leftover stub files/directories
that block `rmdir`. Confirm it's trivial and not real data before touching
it:

```
du -sh /mnt/user/<affected-share> ...
```

If that comes back at a few KB (lock files, partial config, nothing
resembling actual application data, which lives on the real pool/array
behind the share), it's safe to clear:

```
rm -rf /mnt/user/<affected-share> ...
```

`emhttpd`'s retry loop will succeed on its own within one cycle after
that, no need to click Stop again.

## TL;DR

- Docker enabled + zero containers after an unclean shutdown, with the
  array/pools all green, points at a share-resolution problem, not a
  Docker or storage problem.
- Check Unassigned Devices for a drive reporting 0 B / an unsupported
  sector size. It doesn't need to be part of any pool to break share
  resolution for the whole box.
- Removing the bad drive is necessary but not sufficient. Follow it with
  Stop Array → Start Array to actually clear it from the daemon's live
  state.
- If Stop Array hangs on unmounting user shares and `lsof`/`ps` show
  nothing holding it open, check for leftover stub content written
  directly onto the bare `/mnt/user` tmpfs mountpoint before clearing it.
