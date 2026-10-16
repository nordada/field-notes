# field-notes

Root-caused writeups for problems that were annoying enough to solve properly
and specific enough that the fix isn't findable anywhere else. Mostly
homelab/self-hosting infrastructure, occasionally other technical domains.

Each note follows the same shape: what broke, what it actually turned out to
be (not just what fixed it), and the concrete steps to fix and verify it.
Error strings are kept verbatim wherever possible so they're searchable.

## Notes

- [Unraid: Docker shows enabled with zero containers after an unclean shutdown](notes/unraid-docker-empty-after-unclean-shutdown.md). One incompatible drive stalls the share-resolution daemon; pulling the drive isn't enough to clear it.
- [Unraid: "invalid key" plus a fresh-setup screen on a licensed box](notes/unraid-boot-flash-recovery.md). The boot flash dropped off the USB bus and left a zombie mount. The config is intact; do not accept the onboarding screen.
- [Grocy: "Configured AUTH_CLASS does not exist" after a 4.x update](notes/grocy-auth-class-namespace-fix.md). A one-line config fix, and the skipped migrations that pile up behind a boot-time config failure.
- [Git: a polling builder whose fetch timeout equals its poll interval leaks tens of GB](notes/git-polling-builder-tmp-pack-leak.md). 1,512 abandoned `tmp_pack_*` files from fetches killed on the tick, and an auto-gc that could never finish.

- [SteamOS on a desktop board wakes itself seconds after suspend: the chipset PCIe bridge](notes/steamos-desktop-wakes-itself-after-suspend-gpp0.md). The root port to the chipset raises the wake; denying it in `/proc/acpi/wakeup` at boot holds sleep and keeps wake-on-LAN working. With the NetworkManager profile trap, a passwordless remote suspend, and a "Wake" entry for the client's Steam library.
- [Red Dead Redemption 2 first-time setup hangs on "Starting launch" over Remote Play](notes/rdr2-remote-play-starting-launch-hidden-installer-dialog.md). The Rockstar installer ignores its silent flags and waits on a language dialog that gamescope never shows; XSendEvent clicks from a shell get through, XTEST ones do not.

## License

Content is licensed under [CC BY 4.0](LICENSE). Reuse and adapt freely, attribution appreciated.
