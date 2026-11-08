# `sdb` server deployment

This repository runs the combined `sdb` website: public SED results and fitting,
the `sdb` review interface, authentication, and optional static publications.
It is deployment glue rather than application code. The `sdb`, `sdf`, and
`tldr_pacs` projects remain independently installable and versioned.

Apache listens on one public-facing port and routes:

- `/`, `/search`, `/target/...`, and `/system/...` to `sdf-fits serve`;
- `/samples` and `/admin` to the same `sdf` service;
- `/review/` to `sdb review serve`;
- `/docs/sdf/` and `/docs/sdb/` to generated project documentation; and
- `/herschel/` directly to the generated TLDR PACS publication.

Administrators can use `/admin/activity` for catalogue inventory, recent
audited changes, queue health, and aggregate local page-use statistics. The
frontend access log includes user-agent and request-duration fields so the
report can exclude known crawlers and show request timing. The report never
displays or copies client addresses; Cloudflare Web Analytics remains the
appropriate view for visits, referrers, geography, and browser performance.

## Configure a host

Copy the example and edit all paths for the machine:

```shell
cp config/server.conf.example config/server.conf
```

`config/server.conf` is deliberately ignored by Git. The default credential
locations are also ignored:

```text
.htpasswd
session-secret
```

Create `.htpasswd` with Apache's `htpasswd` utility. `session-secret` is a
stable random value shared by the `sdf` and `sdb` processes for signing login
sessions. Keep both private and back them up outside this repository.

The `sdb` database stores registered users, administrator roles, sample
ownership, public/private visibility, and explicit sharing. A password-file
identity can be registered with:

```shell
sdb --database /path/to/sdb.sqlite user add collaborator@example.com
```

Use `user set-admin`, `user remove-admin`, `user disable`, and `user enable` to
change roles and access. Use `sample set-access`, `sample share`, and
`sample unshare` to manage existing samples.

## Run it

```shell
bin/start
bin/status
bin/stop
```

`bin/start` rebuilds both documentation sites with warnings treated as errors.
They can also be rebuilt without restarting the services:

```shell
bin/build-docs
```

For a foreground process suitable for a service manager, use `bin/run`.
Ctrl-C stops Apache and both application services.

On a macOS server, install the supplied per-user launchd service instead of
starting it by hand:

```shell
bin/install-service
```

It starts the complete site at login, restarts it after an unexpected exit,
monitors all three website processes, and writes supervisor output to
`logs/service.log`. Use `bin/status` for the
application processes and `launchctl print gui/$UID/com.drgmk.sdb-server` for
the supervisor. `bin/uninstall-service` stops and removes the service.

Large UI sample imports report their current target/provider stage on the job
page. Ordinary `sdb` commands retain a 30-minute `SDB_IMPORT_TIMEOUT` guard.
Resumable sample imports use `SDB_SAMPLE_IMPORT_TIMEOUT`, which defaults to 12
hours so a healthy large import is not killed merely for taking time. Both can
be changed in `config/server.conf`.

The local development configuration uses two fit workers. On the 10-core,
16 GB M2 Pro this is conservative while Chrome, VS Code, and Codex are also
running. Four workers is a reasonable starting point when the machine is
dedicated to serving and fitting. Active MultiNest workers have previously
used roughly 1.1--1.4 GB RSS each, so increase this only after monitoring peak
memory under representative fits.

## Backups

Install the per-user launchd job once on each macOS server:

```shell
bin/install-backup
```

It runs at 03:15 each day and once when loaded. `bin/backup` can also be run
directly. By default, 14 verified snapshots are retained under
`private/backups`; each contains the live `sdb.sqlite`, the authoritative
spectral assets, `sdb.toml`, the deployment configuration, `.htpasswd`, and
the session secret. The directory is private and ignored by Git.

Set `SDB_BACKUP_ROOT` in `config/server.conf` to a directory on another disk
for protection against disk loss. A local snapshot only protects against
database corruption and operator error. The reference and cache databases are
rebuildable; set `SDB_BACKUP_INCLUDE_REFERENCE=1` or
`SDB_BACKUP_INCLUDE_CACHE=1` if their download cost justifies the extra space.
The large exported fit tree is not included in daily snapshots and is best
handled separately by an incremental, encrypted off-machine backup.

Inspect recent runs with:

```shell
tail -100 logs/backup.log
launchctl print gui/$UID/com.drgmk.sdb-backup
```

## Generated and private state

The following are intentionally not versioned:

- exported SED packages and `sdb` databases;
- `.htpasswd` and `session-secret`;
- `config/server.conf`;
- logs, PID files, and matplotlib caches; and
- `www/`, including generated documentation and the Herschel/PACS publication.

This keeps a clone small and reproducible. Restoring a server consists of
installing `sdb` and `sdf`, supplying the local config and credentials,
restoring the database/export data, rebuilding or copying static publications,
and running `bin/start`.

## Herschel/PACS publication

Build the publication with `tldr_pacs`; it is too large and too reproducible to
store here:

```shell
cd /path/to/tldr_pacs
python web/build.py \
  --product-root /path/to/tldr_pacs_jsmap_annulus_dc \
  --output /path/to/sdb-server/www/herschel \
  --base-url /herschel/ \
  --preview-dpi 150 \
  --preview-timeout 30
```

The `sdb` catalogue provider should use the resulting `manifest.json` and
`/herschel/` as its public base URL.

## Production

The supplied configuration is intended for a loopback service, optionally
published through a trusted tunnel or reverse proxy. For a conventional public
deployment, add HTTPS and host-name configuration at that boundary and run
`bin/run` under the host's service manager. On macOS, `bin/install-service`
provides that launchd integration.
