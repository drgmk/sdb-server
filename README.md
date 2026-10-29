# SDB server deployment

This repository runs the combined SDB website: public SDF results and fitting,
the SDB review interface, authentication, and optional static publications.
It is deployment glue rather than application code. The `sdb`, `sdf`, and
`tldr_pacs` projects remain independently installable and versioned.

Apache listens on one public-facing port and routes:

- `/`, `/search`, `/target/...`, and `/system/...` to `sdf-fits serve`;
- `/samples` and `/admin` to the same SDF service;
- `/review/` to `sdb review serve`; and
- `/herschel/` directly to the generated TLDR PACS publication.

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
stable random value shared by the SDF and SDB processes for signing login
sessions. Keep both private and back them up outside this repository.

The SDB database stores registered users, administrator roles, sample
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

For a foreground process suitable for a service manager, use `bin/run`.
Ctrl-C stops Apache and both application services.

The local development configuration uses two fit workers. On the 10-core,
16 GB M2 Pro this is conservative while Chrome, VS Code, and Codex are also
running. Four workers is a reasonable starting point when the machine is
dedicated to serving and fitting. Active MultiNest workers have previously
used roughly 1.1--1.4 GB RSS each, so increase this only after monitoring peak
memory under representative fits.

## Generated and private state

The following are intentionally not versioned:

- exported SED packages and SDB databases;
- `.htpasswd` and `session-secret`;
- `config/server.conf`;
- logs, PID files, and matplotlib caches; and
- `www/`, including the generated Herschel/PACS publication.

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

The SDB catalogue provider should use the resulting `manifest.json` and
`/herschel/` as its public base URL.

## Production

The supplied configuration is intended for a loopback service, optionally
published through a trusted tunnel or reverse proxy. For a conventional public
deployment, add HTTPS and host-name configuration at that boundary and run
`bin/run` under the host's service manager.
