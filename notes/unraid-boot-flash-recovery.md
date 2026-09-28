# Unraid: "invalid key" plus a fresh-setup screen on a licensed box usually means the boot flash dropped off the USB bus

**If a previously licensed Unraid server shows an "invalid key" error and a
fresh-setup onboarding screen, do not accept anything on that screen.** In
both cases seen so far the configuration was never lost, and the fix was to
reseat a USB stick and reboot.

## The symptom set

All of these appear together, which is the tell:

- Invalid or missing licence key, even though the key downloads fine from
  Unraid Connect. The file is named for the licence tier, so it will be
  `Lifetime.key`, `Pro.key`, `Plus.key` or similar. Do not assume one name.
- Web UI unreachable, ports 80 and 443 refusing connections.
- The box offering to configure a **new** server.
- SSH keys rejected that worked earlier the same day.
- SMB shares gone, port 445 closed.
- Docker containers all down.

## Why they all happen at once

Every one of those lives on the USB boot flash, under `/boot/config`: the
licence key, `super.dat` (the disk assignments), `ssh/`, share configs,
Docker settings. Unraid reads that at boot and holds it in RAM.

**If `/boot` becomes unreadable, Unraid honestly reports what it can see,
which is nothing.** It is not lying and it is not corrupt: it has no config,
so it offers to make one. That is why the screen looks catastrophic while
the data is untouched.

The specific failure is a **zombie mount**. The flash drops off the USB bus,
Linux tears down its device node, but the `/boot` mountpoint keeps pointing
at a device that no longer exists. The stick then usually re-enumerates
under a *different* letter, perfectly healthy, while `/boot` stays attached
to the dead one.

## Triage, in order (all read-only)

**1. Is the flash still physically present, under a new name?**

```
mount | grep ' /boot '
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL
```

If the device `/boot` is mounted from does **not** appear in `lsblk`, but
some other small vfat volume labelled `UNRAID` does, that is a
re-enumeration. **The stick is alive and the data is almost certainly
intact.** This single check is the difference between "reseat and reboot"
and "buy a new stick and do a key transfer".

**2. Confirm the config really is unreadable**

```
ls -A /boot/config | wc -l
dmesg | grep -i -E 'FAT-fs|I/O error|usb'
```

Zero entries plus a stream of `FAT-fs (sdX1): Directory bread ... failed`
confirms the dead mount.

**3. A useful accidental signal**

If something fails with `mkdir: cannot create directory '/root/.ssh': File
exists`, that is diagnostic, not noise. **`mkdir -p` cannot fail on a
directory that already exists**, only on a path that exists and is not one.
`/root/.ssh` is a symlink to `/boot/config/ssh/root`, so this means it is
dangling, which means the flash is unreadable. `ssh-copy-id` fails this way
too.

**4. Do not trust the array state readout**

`/var/local/emhttp/var.ini` stops being updated the moment emhttp loses the
flash, so it reports whatever was true at the instant of failure. In one
incident it claimed `mdState="STARTED"` while nothing was mounted. Check the
live mount table instead:

```
mount | grep -E '/mnt/(disk|user|cache)'
stat -c %y /var/local/emhttp/var.ini    # compare against `date`
```

## Recovery

**1. Back the config up before touching anything.** Mount the re-enumerated
device **read-only** and pull a copy off the box. Do this even if Unraid
Connect has a recent flash backup: this one is current, local, and costs
seconds. Read-only matters because a stick that just dropped off the bus
should not be written to, and because Unraid must not be given a writable
`/boot` while it believes it is onboarding a fresh system, or it may write a
new config straight over the good one.

```
mkdir -p /tmp/fc
mount -o ro,noatime /dev/sdX1 /tmp/fc
ls -l /tmp/fc/config/*.key /tmp/fc/config/super.dat
```

**`*.key` can match more than one file.** A stale `Trial.key` can sit
alongside the real tier key. Check the dates, and confirm the tier key is
the one present.

Do the mount and the pull as a **single** SSH invocation rather than two
steps. While `/boot` is dead, key-based SSH is rejected, so every separate
SSH call means another password prompt.

Then stream it off the machine (run from a workstation):

```
ssh root@<server> 'tar -C /tmp/fc -cf - config' > ~/flash-backup.tar
```

Verify the archive is not truncated and contains the three files that
matter, rather than trusting that it succeeded:

```
tar -tf ~/flash-backup.tar > /dev/null && echo intact
tar -tvf ~/flash-backup.tar | grep -E 'config/([A-Za-z]+\.key|super\.dat|ssh/root/authorized_keys)$'
```

**2. Fix it physically.** Power down (`powerdown -r` for a reboot), open the
case, reseat the stick.

**3. Boot.** The flash enumerates cleanly, `/boot` mounts properly, Unraid
reads the intact key and `super.dat`, and everything returns.

## What you do NOT need (in the reseat case)

- **A key transfer.** The licence binds to the flash's GUID. Same physical
  stick means the same GUID, so the existing key stays valid.
- **The Unraid Connect flash backup.** Useful insurance, unnecessary when
  the config is still on the stick.
- **A New Config or manual disk assignment.** This is the dangerous one:
  assigning disks by hand risks putting a data disk in a parity slot, which
  rebuilds parity *over* real data. `super.dat` already holds the correct
  assignments. **Never accept the onboarding screen's offer while a
  readable `super.dat` still exists.**

## If the stick is actually dead

The `lsblk` check in triage step 1 is what separates this from the reseat
case. If no vfat volume labelled `UNRAID` appears anywhere, the stick has
genuinely failed and you need a new one plus a key transfer.

**The backed-up `.key` file is what makes that possible.** The licence binds
to the flash GUID, and the GUID is embedded in the key file. A dead stick
takes its GUID with it, so the local flash backup is not merely a config
convenience. It is the only copy of the identity a transfer needs.

Outline:

1. **New stick.** Unraid requires a unique GUID, and many cheap no-name or
   very small drives report generic or duplicated GUIDs and are rejected
   outright. 4 to 32 GB, USB 2.0 or 3.0.
2. **Prepare** it with the Unraid USB Creator, then restore `config/` from
   the backup tar over the top.
3. **Key transfer** through Unraid's own registration flow, which reads the
   old GUID from the existing key. **Self-service transfers are limited to
   roughly one per 12 months**, so do not spend one casually.
4. `super.dat` from the backup carries the disk assignments, so the array
   comes back without hand-assignment. The warning above still applies:
   never assign disks manually while a good `super.dat` exists.

Untested. Written from two incidents twelve days apart, both of which turned
out to be the reseat case.

## Expect a parity check afterwards

With `/boot` dead, Unraid physically cannot write its clean-shutdown marker,
so the next boot reports an unclean shutdown and starts a parity check. That
is normal here, not a second fault. It runs for hours, the box stays usable,
load average will look alarming.

## Prevention

**Reseating alone does not hold.** The first reseat lasted nine days before
the same stick worked loose again, with an identical presentation. USB-A
contacts retain by friction, and once a connector has been worked loose and
reseated its retention springs have taken a set, so the second failure comes
faster than the first. **The load has to come off the pins mechanically.**
Reseating re-establishes contact and changes nothing about the cause.

**Mount the dongle so gravity is not working on the connector.** The first
failure was a stick mounted horizontally in an internal motherboard header
whose own weight had slowly worked it loose over months; adding cables to
the rack supplied the last bit of vibration.

An internal header is otherwise the right place for it (it cannot be
knocked out or pulled). Options: a vertically oriented header, a right-angle
adapter, or physically securing the stick so its weight is not carried by
the pins.

**The vertical header is the answer.** Two other boxes of identical
architecture and comparable uptime in the same rack both sit on vertical
headers, where the stick's weight runs along the connector axis instead of
cantilevering off it. Neither has ever had this failure. The one horizontal
mount in the fleet failed twice in twelve days. That is as close to a
controlled experiment as a homelab gets. A foam block under the stick is a
workable stopgap, but foam creeps under sustained load at elevated
temperature, so re-check it and replace it with a rigid cradle or a
relocation to a vertical header.

**After any physical work in the rack, sanity-check the boxes.** The
disconnect happened overnight and was not noticed until the morning.

## Detection

Both failures ran undetected for a long time, which is the real cost. The
second one dropped mid-afternoon and was **found three days later**.

**The signal is already there and nobody is watching it.** Key-based SSH
from a workstation stops working the instant `/boot` dies, because
`/root/.ssh` is a symlink into `/boot/config/ssh/root`. On the second
incident the login banner read "There were 20 failed login attempts since
the last successful login."

To pin the drop time after the fact, compare the mtime of
`config/ssh/root/authorized_keys` (the last moment the flash was writable)
against the first failed login. That bracketed the second incident to about
an hour.

An external liveness check from another machine, alerting when the box
stops answering on 80 and 443, would have caught both in minutes. Unraid's
own notifications cannot help, since the flash they depend on is the thing
that died.
