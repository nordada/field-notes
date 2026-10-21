# SteamOS on a desktop board wakes itself seconds after suspend: the chipset PCIe bridge

## Symptom

A desktop PC running SteamOS (3.8, kernel 6.18) on a Gigabyte B550M DS3H
with a Ryzen 7 3800X, a Radeon RX 570 and the board's Realtek RTL8111 port
(`r8169`) goes to sleep cleanly, screen off, fans stop, and is back on
three to five seconds later. Every time, whether the suspend was started
from Steam's power menu, a short press of the power button, or
`systemctl suspend`.

```
PM: suspend entry (deep)
ACPI: PM: Waking up from system sleep state S3
xhci_hcd 0000:02:00.0: xHC error in resume, USBSTS 0x401, Reinit
r8169 0000:05:00.0 enp5s0: Link is Down
PM: suspend exit
```

The goal was to have it sleep when idle and wake on a wake-on-LAN packet,
so the obvious suspects were the things that had just been enabled for
that.

## What the counters said

The kernel keeps a per-device count of wake events:

```
for w in /sys/class/wakeup/wakeup*; do
  c=$(cat $w/event_count); [ "$c" != 0 ] && echo "$(cat $w/name) events=$c"
done
```

Two counters went up by one on every cycle:

```
device:16 events=2
0000:00:01.1 events=2
```

`0000:00:01.1` is the PCIe root port that leads to the B550 chipset, called
`GPP0` in the firmware's wake table:

```
grep enabled /proc/acpi/wakeup
GP12   S4  *enabled  pci:0000:00:07.1
GP13   S4  *enabled  pci:0000:00:08.1
XHC0   S4  *enabled  pci:0000:08:00.3
GPP0   S4  *enabled  pci:0000:00:01.1
GPP8   S4  *enabled  pci:0000:00:03.1
PTXH   S4  *enabled  pci:0000:02:00.0
PT20   S4  *enabled  pci:0000:03:00.0
PT29   S4  *enabled  pci:0000:03:09.0
```

Everything that could plausibly wake the box sat behind that bridge: the
chipset's USB controller (`PTXH`, with a keyboard and a Logitech receiver
on it), a spare Mellanox network card with no cable (`PT20`), and the
wired port on its own bridge (`PT29`). The counters do not say which one
of them fired.

Two other counters also climbed, `LNXPWRBN:00` and `PNP0C0C:00`, the ACPI
power button devices, by about three per cycle even on suspends started
from the shell with no button involved. That is firmware attribution, not
a stuck button; it was a red herring.

## Elimination

Each test was: change one thing for this boot only, suspend, see whether it
held. `/proc/acpi/wakeup` toggles a device's wake permission when you write
its name to it, as root, and `/sys/class/net/<port>/device/power/wakeup`
does the same for the network port. The kernel log afterwards gives the
exact sleep and wake times:

```
journalctl -k -b -o short-iso | grep -aiE "suspend entry|Waking up from"
```

| Change | Result |
| --- | --- |
| Keyboard and receiver unplugged, Mellanox card pulled | woke after 3 s |
| Wired port's wake flag `disabled`, no wake-on-LAN bits set | woke after 3 s |
| Wired port set to magic packet only | woke after 5 s |
| Both USB controllers (`PTXH`, `XHC0`) denied in `/proc/acpi/wakeup` | woke after 3 s |
| The port's own bridge (`PT29`) denied | woke after 3 s |
| The chipset bridge (`GPP0`) denied | **stayed asleep**, and woke on a magic packet 90 s later |

So the chipset bridge itself raises the wake, not a device behind it, and
denying it does not stop a wake-on-LAN packet addressed to the port behind
it from working. Why the bridge asserts a wake is a firmware question this
note does not answer; denying it is the known workaround on several AMD
boards, and it held here through a reboot and repeated cycles.

## Fix

`/proc/acpi/wakeup` resets on every boot, so the write has to be repeated.
SteamOS's root filesystem is read-only but `/etc` is writable, which is
enough for a unit:

```
# /etc/systemd/system/disable-gpp0-wake.service
[Unit]
Description=Stop the chipset PCIe bridge (GPP0) from waking the machine
After=sysinit.target
ConditionPathExists=/proc/acpi/wakeup

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'grep -q "^GPP0.*enabled" /proc/acpi/wakeup && echo GPP0 > /proc/acpi/wakeup; grep "^GPP0" /proc/acpi/wakeup'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now disable-gpp0-wake.service
```

Verify, after a reboot:

```
grep ^GPP0 /proc/acpi/wakeup
GPP0   S4  *disabled  pci:0000:00:01.1
```

then suspend, wait a minute, and send the packet.

## The trap on the way: the wake-on-LAN setting landed on a dead profile

Wake-on-LAN on SteamOS is a NetworkManager connection property:

```
sudo nmcli con modify "Wired connection 1" 802-3-ethernet.wake-on-lan magic
sudo nmcli con up "Wired connection 1"
```

That was done early, and then the spare PCIe card was pulled during the
elimination. Removing a card renumbers the PCI bus, the port's kernel name
changed from `enp5s0` to `enp4s0`, and NetworkManager quietly created a
new profile for the "new" device, "Wired connection 2", with the wake
setting at its default. The port reported `wake-on-lan: magic` for a
profile that was no longer in use. Check which profile is active before
setting anything:

```
nmcli -t -f NAME,DEVICE con show --active
```

SteamOS ships no `ethtool`, so to read the bits the driver actually has
set, which needs root:

```
sudo python3 - enp4s0 <<'PY'
import socket, fcntl, struct, sys, ctypes
SIOCETHTOOL, ETHTOOL_GWOL = 0x8946, 0x5
names = {1:"phy", 2:"unicast", 4:"multicast", 8:"broadcast", 16:"arp", 32:"magic", 64:"magicsecure"}
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
b = ctypes.create_string_buffer(struct.pack("III6s", ETHTOOL_GWOL, 0, 0, b"\0"*6))
fcntl.ioctl(s.fileno(), SIOCETHTOOL, struct.pack("16sP", sys.argv[1].encode(), ctypes.addressof(b)))
_, sup, opts, _ = struct.unpack("III6s", b.raw)
dec = lambda v: ", ".join(n for bit, n in names.items() if v & bit) or "none"
print("supported:", dec(sup)); print("enabled:", dec(opts))
PY
```

With the setting on the right profile this prints `enabled: magic`. The
BIOS side on this board was Settings, Platform Power, ErP Disabled and
Wake on LAN Enable Enabled.

## Sending the packet, and sleeping it, from another machine

A magic packet is six `0xff` bytes followed by the MAC sixteen times, sent
to the LAN broadcast on UDP port 9 (7 also works). Nothing needs
installing:

```
python3 - aa:bb:cc:dd:ee:ff <<'PY'
import socket, sys
mac = bytes.fromhex(sys.argv[1].replace(":", "")); pkt = b"\xff" * 6 + mac * 16
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
for port in (9, 7): s.sendto(pkt, ("255.255.255.255", port))
PY
```

To sleep it over SSH without a password, `systemctl suspend` from a
non-local session needs root, and a one-line sudoers rule for exactly that
command is the narrowest way:

```
deck ALL=(root) NOPASSWD: /usr/bin/systemctl suspend
```

Put it in `/etc/sudoers.d/` with a file name that sorts **after** the
wheel group's rule, for example `zz-deck-suspend`, mode 440, and check with
`visudo -c`. sudo applies the last matching rule, and the drop-ins are read
in name order; named `deck-suspend` it sorted first, lost to the wheel
rule, and kept asking for a password. Then:

```
ssh thepc sudo -n systemctl suspend
```

## Waking it from the couch: a "Wake" entry in the client's Steam library

Steam Remote Play cannot wake a sleeping host, and the client's Steam
stores no MAC for it, only the name. But any script can be a non-Steam
library entry, so the wake is one:

```
#!/bin/bash
# ~/.local/bin/wake-pc.sh on the SteamOS client
MAC=aa:bb:cc:dd:ee:ff; IP=192.0.2.10; PORT=27036
ready() { timeout 1 bash -c "</dev/tcp/$IP/$PORT" 2>/dev/null; }
ready && exit 0
python3 - "$MAC" <<'PY'
import socket, sys
mac = bytes.fromhex(sys.argv[1].replace(":", "")); pkt = b"\xff" * 6 + mac * 16
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
for port in (9, 7): s.sendto(pkt, ("255.255.255.255", port))
PY
start=$SECONDS
while [ $((SECONDS - start)) -lt 90 ]; do sleep 2; ready && exit 0; done
exit 1
```

The wait is on TCP 27036, the port Steam's Remote Play host listens on,
not on ping. The network comes back a moment before Steam does (one second
apart on a resume from suspend, longer on a cold boot), and a stream started
in that gap fails. `/dev/tcp` is bash's own, so nothing needs installing.

In Desktop mode: Steam, Games, Add a Non-Steam Game, Browse. Folders
starting with a dot are hidden in the picker; Alt+. shows them, or type
the full path into the file name box. Rename the entry, run it from Gaming
Mode, wait for it to return, then stream. It returns as soon as the host's
Steam answers, so a quick return means the PC is ready to stream from.

Steam's artwork for a non-Steam entry lives in
`~/.local/share/Steam/userdata/<account>/config/grid/`, named by the
entry's app id. Four files cover every view:

| File | Size | Where it shows |
| --- | --- | --- |
| `<id>p.png` | 600 by 900 | the library grid tile |
| `<id>.png` | 920 by 430 | the recent-games row and other wide views |
| `<id>_hero.png` | 1920 by 620 | the banner across the top of the entry's page |
| `<id>_logo.png` | 800 by 300, transparent | the logo drawn over the hero |

The small icon is separate: set it in the entry's Properties, where the
dialog does not redraw it until Steam restarts, even though it is saved.

What the set looks like, in Steam's own blue and grey:

![icon](assets/wake-pc/icon.png)

![portrait tile](assets/wake-pc/portrait.png)

![wide capsule](assets/wake-pc/wide.png)

![hero with the logo over it](assets/wake-pc/hero-with-logo.png)

To make your own set with ImageMagick:

1. **Find the entry's id.** It is stored in `config/shortcuts.vdf`, a
   binary file; this prints the name and id of every non-Steam entry:

   ```
   python3 - ~/.local/share/Steam/userdata/<account>/config/shortcuts.vdf <<'PY'
   import sys, re
   b = open(sys.argv[1], "rb").read()
   for e in b.split(b"\x02appid\x00")[1:]:
       f = dict(re.findall(rb"\x01([A-Za-z ]+)\x00([^\x00]*)\x00", e))
       print(int.from_bytes(e[:4], "little"), f.get(b"AppName", b"").decode())
   PY
   ```

2. **Render the files**, with the id and a font path of your own. Any
   TrueType or OpenType font works; the one below is a macOS path.

   ```
   ID=2456176968
   F=/System/Library/Fonts/HelveticaNeue.ttc
   BG='#1b2838'; FG='#66c0f4'; TX='#c7d5e0'
   magick -size 512x512  xc:$BG -fill none -stroke $FG -strokewidth 28 -draw "arc 136,116 376,356 -60 240" -draw "line 256,96 256,236"   -stroke none -fill $TX -font "$F" -pointsize 54  -gravity south -annotate +0+58  "WAKE PC" icon.png
   magick -size 600x900  xc:$BG -fill none -stroke $FG -strokewidth 34 -draw "arc 150,250 450,550 -60 240" -draw "line 300,225 300,400"  -stroke none -fill $TX -font "$F" -pointsize 64  -gravity south -annotate +0+150 "WAKE" -annotate +0+70 "PC" ${ID}p.png
   magick -size 920x430  xc:$BG -fill none -stroke $FG -strokewidth 26 -draw "arc 110,95 350,335 -60 240"  -draw "line 230,75 230,215"   -stroke none -fill $TX -font "$F" -pointsize 86  -gravity west  -annotate +420+0 "WAKE\nPC" ${ID}.png
   magick -size 1920x620 xc:$BG -fill none -stroke '#2a475e' -strokewidth 40 -draw "arc 1380,110 1780,510 -60 240" -draw "line 1580,80 1580,310" ${ID}_hero.png
   magick -size 800x300  xc:none -fill $TX -font "$F" -pointsize 110 -gravity center -annotate +0+0 "WAKE PC" ${ID}_logo.png
   ```

   The `arc` is the ring of the power symbol with a gap at the top, the
   `line` is the bar through the gap. Change the text, the colours or the
   font and nothing else needs touching.

3. **Put them in place and restart Steam.** Copy the four `${ID}` files
   into the `grid` folder (create it if it is missing; `scp -r` from
   another machine is fine), set `icon.png` in the entry's Properties, then
   restart Steam, which from Gaming Mode is a switch to Desktop and back or
   the Power menu's Restart Steam. The tile and banner appear on the next
   library view.

## Verify

1. `grep ^GPP0 /proc/acpi/wakeup` reads `disabled` after a fresh boot.
2. Suspend; the PC stays silent to ping for at least a minute.
3. Send the packet; the kernel logs `Waking up from system sleep state S3`
   within a few seconds and the PC answers ping about ten seconds later.
