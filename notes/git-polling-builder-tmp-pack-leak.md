# Git: a polling builder whose fetch timeout equals its poll interval leaks tens of GB of `tmp_pack_*` files

## Symptom

A static-site builder container polls a git remote every two minutes and
rebuilds on change. The repository it clones is small: the working tree is
about 150 MB and the upstream repo packs into a single 159 MB pack. On the
host, its `.git` directory had grown to **48.9 GB**, nearly half the pool it
lived on, and was still growing at roughly 1.4 GB per day.

Nothing in the container log looked wrong. Builds still succeeded. The site
kept serving.

## What the numbers said

```
git count-objects -vH
```

reported about **10,137 loose objects** and **41 packs** totalling around
6.5 GB. That still did not add up to 48.9 GB. The rest was here:

```
ls .git/objects/pack/ | grep -c '^tmp_pack_'
du -sh .git/objects/pack/tmp_pack_*
```

**1,512 abandoned `tmp_pack_*` files, about 42 GB.** Git writes a pack to a
`tmp_pack_` name while building it and renames it into place only on
success. A `tmp_pack_` that is still there after the operation ended is a
pack that was killed partway through. Their sizes averaged about 28 MB
against a full pack of 159 MB, spread over a wide range, which is the
signature of a process interrupted at random points rather than failing at
one consistent step.

## Root cause

The builder's loop script set two values:

```
POLL_INTERVAL=120
FETCH_TIMEOUT=120
```

and ran each fetch as `timeout "$FETCH_TIMEOUT" git fetch ...`. Any fetch
that needed more than two minutes was killed by `timeout`'s SIGTERM at
exactly the moment the next poll cycle was due to begin. Git does not clean
up a `tmp_pack_` on SIGTERM. So every slow fetch left one behind, and the
next cycle started another.

That alone explains the leak. A second loop sat underneath it and made sure
it never self-corrected:

- Git's `gc.auto` default is 6,700 loose objects. Once the count passed it,
  git tried an automatic repack after operations.
- That repack also ran inside the two-minute window, on a repository whose
  pack directory was by then tens of GB, so it also got killed.
- A killed repack leaves the loose objects loose. The count never drops
  below the threshold, so the repack is attempted again on the next cycle,
  and killed again.

The 41 packs that were never consolidated into one are the evidence that
maintenance had **never once completed** in the life of the container.

The two-minute ceiling was itself a deliberate fix, added after a fetch hung
for over two hours and stalled the builder. The ceiling was the right idea.
The value was set below what the work actually needs, and set equal to the
poll interval, which is the combination that turns one slow fetch into a
permanent leak.

## Why it cost more than disk

The repository directory was included in the nightly application-data
backup, uncompressed. A 49 GB tarball every night is how a backup share
reached 619 GB while the real data it protected was under 200 MB.

## Fix

The repository was a disposable clone that the builder re-creates on start,
so the cheapest correct fix was to throw it away rather than repair it:

1. Stop the builder container.
2. Delete the clone directory. The web container serving the built output
   was untouched, so the site stayed up, frozen at the last build.
3. Exclude the clone directory from the backup job. A disposable clone has
   no business in a backup.

Before starting the builder again, the loop script needs four changes:

- **`FETCH_TIMEOUT` well above the worst-case transfer**, not the typical
  one. The original hang that prompted the timeout was a network fault, and
  a generous ceiling still catches it.
- **`POLL_INTERVAL` larger than `FETCH_TIMEOUT`**, so two attempts can never
  overlap. Equal values guarantee the kill lands at the worst moment.
- **`git config gc.auto 0`** in the clone, with a deliberate `git gc` on a
  schedule instead. Automatic gc inside a timeout-wrapped command is a repack
  that will eventually be killed.
- **Sweep stale `tmp_pack_*` files on startup.** Anything under
  `.git/objects/pack/tmp_pack_*` older than the current run is garbage.

## Verify

After the fix, `git count-objects -vH` on the fresh clone should show a
handful of packs, loose objects in the hundreds at most, and `size-garbage`
of zero. Watch `du -sh .git` across a few days of polling. It should not
move.

## Unverified

Whether the killed operation was the fetch itself or the auto-repack that
follows it was never pinned down. `.git/gc.log` and the container log would
separate the two. It does not change the fix, since both are cured by the
same four changes.
