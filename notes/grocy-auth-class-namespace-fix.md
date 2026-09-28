# Grocy: "Configured AUTH_CLASS does not exist" after a 4.x image update, and the migration debt hiding behind it

**If Grocy refuses to boot with `Invalid setting in config.php: Configured
AUTH_CLASS "Grocy\Middleware\DefaultAuthMiddleware" does not exist`, the
container is fine and the config is stale.** Grocy 4.x moved the auth
middleware classes into a sub-namespace and an older `config.php` in
persistent storage still points at the 3.x location. The config edit itself
is one line, but expect a second failure immediately behind it: while Grocy
was failing this check it was also skipping its database migrations, so the
schema is behind as well. Both halves are below, fix them in order.

Hit 2026-09-09 on `lscr.io/linuxserver/grocy` running Grocy 4.7.1, against a
`config.php` dated 2025-07-21.

## The symptom

Every page returns the same error, and the container itself reports `Up`.
That combination is the tell: PHP is running, Grocy parsed `config.php`
successfully, and it is failing its own validation pass rather than crashing.

```
Invalid setting in config.php: Configured AUTH_CLASS
"Grocy\Middleware\DefaultAuthMiddleware" does not exist
```

`app.php` runs a `class_exists()` check on the configured `AUTH_CLASS` before
booting and bails out via `ConfigurationValidator` when it fails.

## The cause

Grocy 4.x moved every auth middleware down one level:

```
3.x:  Grocy\Middleware\DefaultAuthMiddleware
4.x:  Grocy\Middleware\Auth\DefaultAuthMiddleware
```

The classes now live in `/app/www/middleware/Auth/`, and nothing named
`DefaultAuthMiddleware.php` remains directly under `middleware/`. The same
move applies to `ReverseProxyAuthMiddleware`, `LdapAuthMiddleware`,
`ApiKeyAuthMiddleware` and `SessionAuthMiddleware`.

The reason this survives an image update is that `config.php` lives in the
persistent config volume (`/config/data/config.php`), so it persists across
container rebuilds while the app code underneath it moves on.

## Why only this one setting breaks

`app.php` loads the user config first and the shipped defaults second:

```php
require_once GROCY_DATAPATH . '/config.php';
require_once __DIR__ . '/config-dist.php'; // fallback for anything not set
```

Settings the user never defined are backfilled from `config-dist.php`, so a
config that is merely *old* self-heals for newly added settings. Only values
that are explicitly set **and** whose meaning moved will break. `AUTH_CLASS`
is the one that moved.

## Triage

Read-only, run on the Docker host. Confirms the diagnosis before touching
anything.

```sh
# what the config asks for
docker exec grocy sh -c 'grep -n AUTH_CLASS /config/data/config.php'

# what this image actually ships
docker exec grocy sh -c 'grep -n AUTH_CLASS /app/www/config-dist.php'

# where the classes really are
docker exec grocy sh -c 'grep -Hn "^namespace" /app/www/middleware/Auth/*.php'

# version, to confirm a major bump happened
docker exec grocy sh -c 'cat /app/www/version.json'
```

## The config fix

Edit the single `AUTH_CLASS` line in `/config/data/config.php` to match the
default in that image's own `config-dist.php`, then restart. Back up first.

```sh
Setting('AUTH_CLASS', 'Grocy\Middleware\Auth\DefaultAuthMiddleware');
```

Edit from the host side against the volume's path rather than inside the
container. The LinuxServer image is Alpine based, so `sed` in there is
busybox and does not handle the escaping the same way. Resolve the host path
with:

```sh
docker inspect -f '{{range .Mounts}}{{println .Source .Destination}}{{end}}' grocy
```

Restart the container afterwards and confirm the error is gone from the
response body, not just that the container reports `Up`. `Up` was true the
whole time it was broken.

## Second act: the schema is behind too

Once auth resolves, the next thing you hit is a failed login:

```
SQLSTATE[HY000]: General error: 1 table sessions has no column named token_type
```

This is not a new problem, it is the same one wearing a different hat.
`app.php` validates the config and calls `exit()` on failure **before** it
reaches the migration trigger, so every boot during the AUTH_CLASS outage
also skipped migrations. In this case the database sat at version 255 while
the shipped set went to 257. Migration `0256.sql` is the one that rebuilds
`sessions` with `token_type`.

### Why it does not fix itself on restart

Grocy decides whether to migrate by hashing `version.json` plus
`GROCY_BASE_URL` and `GROCY_BASE_PATH`, then looking for a marker file named
after that hash in `data/viewcache/`. On a miss it empties the viewcache,
**touches the marker, resets opcache, and redirects to `/`**, because the
migration itself runs in `SystemController::Root`.

The marker is written *before* the migration runs, which makes it a one-shot
latch. If that redirect is never followed through to the root route, the
marker already exists and the branch never fires again. The instance then
sits indefinitely on an unmigrated schema with no further attempt.

### Applying the migrations

Load the **root** URL, not `/login`. `BaseAuthMiddleware` treats the `root`
and `login` routes as public, so `/` runs unauthenticated and applies the
pending migrations on the way through. A single page load is enough.

**Back up `grocy.db` first.** Migrations here are destructive by design:
`0256` drops and recreates `sessions`, which invalidates every active login.
Stop the container before copying the file so the snapshot is not taken
mid-write.

### Verify

```sh
docker exec -i grocy php -f /dev/stdin <<'PHP'
<?php
$db = new PDO('sqlite:/config/data/grocy.db');
$c = $db->query("SELECT COUNT(*) c, MAX(migration) m FROM migrations")->fetch();
echo "migrations max={$c['m']}\n";
$cols=[]; foreach ($db->query("PRAGMA table_info(sessions)") as $r) $cols[]=$r['name'];
echo "sessions: " . implode(', ', $cols) . "\n";
PHP
```

The `migrations` table column is `execution_time_timestamp`, not
`execution_time`, which is an easy one to guess wrong when querying it.

### Check the blast radius before running it

Worth confirming against the live data rather than assuming, since `0256`
drops a table and `0257` mutates products:

- Nothing should reference `sessions` (no views, triggers or foreign keys),
  and `PRAGMA foreign_keys` is off by default here anyway.
- `0257` nulls self-referencing parent products. Count them first, it is
  usually zero and the statement is then a no-op.
- `api_keys` is a separate table that `ApiKeyService` still reads, so
  existing API keys are not affected by this migration.

## Red herring: there is no vendor directory

`/app/www/vendor/autoload.php` does not exist in this image and its absence
is **not** the problem. LinuxServer ships the Composer tree as
`/app/www/packages/` instead. Autoloading works fine. Do not go chasing a
rebuild of the dependency tree on the strength of that missing path.

## The general lesson

This is the failure mode of any container whose config lives in a persistent
volume and whose image tracks `latest`. The config outlives major version
bumps and no one is watching for the migration. Grocy fails loudly, which is
the good case. The worrying version is a container that silently keeps
running on a stale setting.

Worth deciding per container whether to pin the image tag or to diff the
shipped `config-dist` against the live config after any major bump.

The sharper lesson is the compounding one. A config check that fails early
also suppresses everything that runs after it, migrations included. So the
visible error is not necessarily the whole outage, and clearing it can just
reveal the debt that accumulated behind it. After fixing any boot-time
config failure, check what the app skipped while it was down rather than
assuming a clean start.
