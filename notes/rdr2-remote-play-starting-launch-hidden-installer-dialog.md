# Red Dead Redemption 2 first-time setup hangs on "Starting launch" over Remote Play: the Rockstar installer's hidden language dialog

## Symptom

Red Dead Redemption 2 (Steam app 1174180) installed on a SteamOS PC,
launched for the first time from another SteamOS machine over Steam Remote
Play. The client sits on "Starting launch" indefinitely. On the host, Steam
shows "Performing first time setup" and never gets past it. Nothing is
downloading, the game is fully installed, the disk is not full.

On the host the setup is visibly stuck on one process:

```
pgrep -af "exe"
... S:\legacycompat\iscriptevaluator.exe legacycompat\evaluatorscript_1174180.vdf
... S:\legacycompat\SteamService.exe /installscript S:\steamapps\common\Red Dead Redemption 2\runasadmin.vdf 1174180
... S:\steamapps\common\Red Dead Redemption 2\Redistributables\Rockstar-Games-Launcher.exe /s /t
```

and the installer's own log, inside the game's Proton prefix, ends with it
waiting for a choice nobody can make:

```
compatdata/1174180/pfx/drive_c/ProgramData/Rockstar Games/Launcher/installer_log.txt

[Info] Installer Version: 1.0.109.3031
[Info] Normal install mode
[Info] Load Init Page
[Info] Unload Init Page
[Info] Load Global Page
[Info] Load Language Page
[Info] Set Language:
```

## Cause

Steam's install script runs the Rockstar Games Launcher installer with
`/s /t`, which are supposed to make it silent. This version of the
installer (1.0.109.3031) logs "Normal install mode" anyway and opens its
Language select dialog.

Under gamescope, Proton apps run on a second X server, `DISPLAY=:1`. The
dialog is there, mapped and waiting:

```
DISPLAY=:1 xwininfo -root -tree | grep '"'
0x2600001 "Language select": ("steam_proton" "steam_proton")  1028x600+0+0  +0+0
```

but gamescope keeps the Steam interface in front while first-time setup
runs, so neither the host's monitor nor the stream ever shows it, and no
input reaches it:

```
DISPLAY=:0 xprop -root GAMESCOPE_FOCUSED_APP
GAMESCOPE_FOCUSED_APP(CARDINAL) = 769
```

769 is Steam itself. The installer waits forever, Steam's install script
waits for the installer, and the client waits for Steam.

## Fix

Drive the dialog from a shell on the host (SSH is fine). SteamOS ships
`xdotool`. Two things matter and cost the most time to find:

1. **Synthetic XTEST input does not reach this dialog.** Plain
   `xdotool click` and `xdotool key` do nothing. A window-targeted click,
   which goes by XSendEvent, does:

   ```
   export DISPLAY=:1
   xdotool mousemove --sync 913 533      # position the pointer
   xdotool click --window 0x2600001 1    # deliver the click to that window
   ```

2. **The installer's log tells you which page you are on**, so you can
   work blind:

   ```
   grep -a "Load .* Page" ".../ProgramData/Rockstar Games/Launcher/installer_log.txt" | tail -n 1
   ```

For the 1028 by 600 dialog, with English preselected, the pages and the
click positions that advance them:

| Page in the log | Click |
| --- | --- |
| Language | Continue at 913,533 |
| Welcome | Continue at 913,533 |
| Legal | the "I accept" box at 107,480, then Continue at 913,533 |
| Directory | optionally untick the desktop shortcut at 109,370, then Continue at 913,533 |
| Install | wait |
| Finished | Close at 913,533 |

When the log shows `Load Finished Page` and the window is gone, the
launcher exists at `pfx/drive_c/Program Files/Rockstar Games/Launcher`,
Steam's setup completes within seconds, and the game launches. Later
launches skip the installer entirely.

To see what you are clicking, there is no `xwd` on SteamOS, but Python can
pull a window's pixels through libX11 and ImageMagick turns the raw dump
into a PNG on any machine:

```
python3 - :1 0x2600001 1028 600 /tmp/dialog.rgb <<'PY'
import ctypes, sys
X = ctypes.CDLL("libX11.so.6")
X.XOpenDisplay.restype = ctypes.c_void_p; X.XOpenDisplay.argtypes = [ctypes.c_char_p]
X.XGetImage.restype = ctypes.c_void_p
X.XGetImage.argtypes = [ctypes.c_void_p, ctypes.c_ulong, ctypes.c_int, ctypes.c_int, ctypes.c_uint, ctypes.c_uint, ctypes.c_ulong, ctypes.c_int]
X.XGetPixel.restype = ctypes.c_ulong; X.XGetPixel.argtypes = [ctypes.c_void_p, ctypes.c_int, ctypes.c_int]
disp, win, w, h, out = sys.argv[1], int(sys.argv[2], 16), int(sys.argv[3]), int(sys.argv[4]), sys.argv[5]
d = X.XOpenDisplay(disp.encode()); img = X.XGetImage(d, win, 0, 0, w, h, 0xffffffff, 2)
buf = bytearray(w * h * 3); i = 0
for y in range(h):
    for x in range(w):
        p = X.XGetPixel(img, x, y); buf[i:i+3] = ((p >> 16) & 255, (p >> 8) & 255, p & 255); i += 3
open(out, "wb").write(buf)
PY
magick -size 1028x600 -depth 8 rgb:/tmp/dialog.rgb dialog.png
```

The same two facts apply to the launcher's own prompts later, such as the
Cloud Saves question on first run: they are drawn on `:1` and may not take
input from the stream, and a click aimed at the window from the host's
shell gets through.

## Verify

1. `ls "pfx/drive_c/Program Files/Rockstar Games/"` lists `Launcher` beside
   `Social Club`.
2. `pgrep -f Rockstar-Games-Launcher.exe` finds nothing.
3. The client's "Starting launch" turns into the Rockstar launcher window,
   then the game.
