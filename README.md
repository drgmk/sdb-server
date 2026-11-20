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

If an application backend is briefly unavailable, Apache serves a small static
error page from `config/backend-unavailable.html`. It preserves and polls the
original URL so an import progress page recovers automatically after a service
restart. This fallback applies to HTTP 502, 503, and 504 responses and does not
depend on either Python service being available.

Administrators can use `/admin/activity` for catalogue inventory, recent
audited changes, queue health, and aggregate local page-use statistics. The
frontend access log includes user-agent and request-duration fields so the
report can exclude known crawlers and show request timing. The report never
displays or copies client addresses; Cloudflare Web Analytics remains the
appropriate view for visits, referrers, geography, and browser performance.

Set `SDF_ACTIVITY_TIMEZONE` to an IANA timezone such as
`Pacific/Auckland` to control the event times and daily page-view dates shown
on the activity page. Timestamps remain stored in UTC.

## Configure a host

Prepare a production-data tree before configuring the services:

```shell
bin/init-data-root --dry-run
bin/init-data-root
```

This only creates directories; it does not activate them or alter an existing
server. Optional CASSIS and Herschel trees can be copied with explicit flags.
See [the production data guide](docs/production-data.md) for the layout,
durability classes, migration cautions, and `tldr_pacs` integration.

Copy the example and edit all paths for the machine:

```shell
cp config/server.conf.example config/server.conf
```

SDF's complete defaults are installed from ``sdf/sdf.toml``. Put only local
overrides in `~/.sdf.conf`; a server normally just selects its full model
library:

```ini
[file]
model_root = /path/to/full/model/library/
```

The website, workers, cleanup job, documentation build, and standalone
`sdf-fit` all load the same defaults and user override automatically.

`config/server.conf` is deliberately ignored by Git. Credential locations are
configured there and should remain private:

```text
/path/to/sdb-data/private/.htpasswd
/path/to/sdb-data/private/session-secret
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

## Sampler-work cleanup

Completed fits retain compact posterior, analysis, and plotting artifacts.
The much larger native MultiNest files under each `seds/*/.work/` directory are
only useful temporarily for resuming interrupted work or diagnosing a recent
fit. Inspect the current cleanup decision without changing files:

```shell
sdf-fits cleanup "$SDF_EXPORT_ROOT"
```

Install the daily launchd job with:

```shell
bin/install-cleanup
```

It runs at 04:15 local time and applies the retention values in
`config/server.conf`: seven days for superseded completed work and 30 days for
committed or interrupted work. Queued/running packages and held fit locks are
always skipped. The installer does not run cleanup immediately. Inspect runs
with:

```shell
tail -100 logs/cleanup.log
launchctl print gui/$UID/com.drgmk.sdb-cleanup
```

Use `bin/uninstall-cleanup` to unload and disable the job.

## Backups

Install the per-user launchd job once on each macOS server:

```shell
bin/install-backup
```

It runs at 03:15 each day and once when loaded. `bin/backup` can also be run
directly. The production example retains 14 verified snapshots under
`$SDB_DATA_ROOT/backups/daily`; each contains the live `sdb.sqlite`,
`sdf-operator.sqlite` (persistent fit customisations and job state), the
authoritative spectral assets, `sdb.toml`, the deployment configuration,
`.htpasswd`, and the session secret. The optional `~/.sdf.conf` override is
included as well. Both SQLite databases are copied online and integrity-checked.
The directory is owner-only and outside Git.

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
- `www/`, including generated documentation; the production example keeps the
  optional Herschel/PACS publication under the external data root.

This keeps a clone small and reproducible. Restoring a server consists of
installing `sdb` and `sdf`, supplying the local config and credentials,
restoring the database/export data, rebuilding or copying static publications,
and running `bin/start`.

## Herschel/PACS publication

Build the optional publication with
[`tldr_pacs`](https://github.com/drgmk/tldr_pacs); it is too large and too
reproducible to store here:

```shell
cd /path/to/tldr_pacs
python web/build.py \
  --product-root /path/to/tldr_pacs_jsmap_annulus_dc \
  --output /path/to/sdb-data/public/herschel-next \
  --base-url /herschel/ \
  --preview-dpi 150 \
  --preview-timeout 30
```

The publisher requires a new output directory. Inspect it, publish it as the
configured `HERSCHEL_ROOT`, then point the `sdb` catalogue provider at the
resulting `manifest.json` with `/herschel/` as its public base URL. Servers that
do not use TLDR PACS can omit this integration.

## Production

The supplied configuration is intended for a loopback service, optionally
published through a trusted tunnel or reverse proxy. For a conventional public
deployment, add HTTPS and host-name configuration at that boundary and run
`bin/run` under the host's service manager. On macOS, `bin/install-service`
provides that launchd integration.
