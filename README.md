# field-notes

Root-caused writeups for problems that were annoying enough to solve properly
and specific enough that the fix isn't findable anywhere else. Mostly
homelab/self-hosting infrastructure, occasionally other technical domains.

Each note follows the same shape: what broke, what it actually turned out to
be (not just what fixed it), and the concrete steps to fix and verify it.
Error strings are kept verbatim wherever possible so they're searchable.

## Notes

- [Unraid: Docker shows enabled with zero containers after an unclean shutdown](notes/unraid-docker-empty-after-unclean-shutdown.md) — one incompatible drive stalls the share-resolution daemon; pulling the drive isn't enough to clear it.

## License

Content is licensed under [CC BY 4.0](LICENSE) — reuse and adapt freely, attribution appreciated.
