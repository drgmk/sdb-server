# Production data layout

Keep mutable catalogue state and generated products outside the `sdb`, `sdf`,
and `sdb-server` Git repositories. The repositories should be replaceable code;
the data root is the long-lived instance.

The recommended default is `~/sdb-data`:

```text
sdb-data/
├── db/
│   └── sdb.sqlite
├── catalogs/
│   ├── sdb-cache.sqlite
│   ├── sdb-cache.sqlite.catalogs/
│   ├── sdb-reference.sqlite
│   └── sdb-reference.sqlite.catalogs/
├── assets/
│   └── spectra/
├── imports/
│   ├── cassis/                         # optional source archive
│   └── herschel/<pipeline-product>/    # optional TLDR PACS inputs
├── models/
│   └── sdf/
├── seds/
│   ├── sdf-operator.sqlite
│   ├── sdf-results.sqlite
│   └── sdbid-v3-.../
├── public/
│   └── herschel/                       # optional generated publication
├── private/
│   ├── .htpasswd
│   └── session-secret
└── backups/
    └── daily/
```

## Prepare an unused data root

The initializer is idempotent and deliberately does not activate the result:

```shell
bin/init-data-root
```

Use another location with `--root` or `SDB_DATA_ROOT`. Preview with
`--dry-run`. Optional existing archives can be merge-copied at setup time:

```shell
bin/init-data-root \
  --root /srv/sdb-data \
  --cassis /data/spitzer/irs/cassis \
  --herschel-source /data/herschel/tldr_pacs_jsmap_annulus_dc \
  --herschel-public /srv/old-site/herschel
```

Copies use `rsync -a` without `--delete`; sources are never moved and extra
destination files are never removed. Source arguments are validated before the
directory tree is changed. `private/` and `backups/` are made owner-only.

The script does **not** create or copy databases, model grids, credentials,
configuration, SED packages, or launchd jobs. It does not edit `server.conf`,
`sdb.toml`, or `~/.sdf.conf`.

## Optional spectra

An installation does not need a bulk spectrum archive. Spectral products can
be imported later through `sdb`; accepted files are copied into the configured
checksum-addressed `assets/spectra/` store and exported into relevant SED
packages. A full CASSIS directory under `imports/` is merely a convenient
source collection for future ingestion. It is not read automatically by the
server.

## Optional Herschel publication

The Herschel catalogue and static browser are produced by
[`tldr_pacs`](https://github.com/drgmk/tldr_pacs). They are optional: `sdb` and
SED fitting work without a `/herschel/` publication.

`tldr_pacs/web/build.py` takes its exported CSV catalogues and a pipeline
product root, then writes a stand-alone static publication. Build into a new
staging directory because the publisher intentionally refuses to overwrite an
existing output:

```shell
git clone https://github.com/drgmk/tldr_pacs.git
cd tldr_pacs
python web/build.py \
  --product-root /srv/sdb-data/imports/herschel/tldr_pacs_jsmap_annulus_dc \
  --output /srv/sdb-data/public/herschel-next \
  --base-url /herschel/
```

After inspecting the staged build, publish it as `public/herschel/` and point
`HERSCHEL_ROOT` there. Do not serve the pipeline-product directory itself.

## Activating the layout

Activation is a separate, deliberate operation. The example `server.conf`
shows the corresponding paths. The principal changes are:

- `SDB_DATABASE` → `db/sdb.sqlite`;
- `SDB_CACHE_DATABASE` and `SDB_REFERENCE_DATABASE` → `catalogs/`;
- `SDB_ASSET_ROOT` → `assets/`;
- `SDF_EXPORT_ROOT` → `seds/`;
- `HERSCHEL_ROOT` → `public/herschel/`;
- credentials → `private/`; and
- `SDB_BACKUP_ROOT` → `backups/daily/`.

Set `assets.root` and `export.root` in the instance's `sdb.toml` to the same
asset and SED paths. Set `[file] model_root` in `~/.sdf.conf` to `models/sdf/`.

Do not raw-copy a live SQLite database. Stop writers or use `sdb backup`/the
SQLite online backup API, update all paths together, then verify database
integrity and every `bin/status` route. Retain the previous data tree until a
successful backup and a representative import/export/fit have completed.

## Durability and backup

Treat these classes differently:

- **Authoritative:** `sdb.sqlite`, `sdf-operator.sqlite`, spectral assets,
  credentials, and any source data that cannot be reacquired.
- **Expensive but reproducible:** completed SED fits, posteriors, model grids,
  and static publications.
- **Rebuildable:** `sdb-cache.sqlite`, `sdb-reference.sqlite`, companion
  catalogue directories, and `sdf-results.sqlite`.
- **Disposable:** sampler `.work` directories, temporary indexes, logs, and
  generated documentation.

The daily `sdb` backup should write to `backups/daily/` and retain the main and
operator databases, assets, credentials, and local configuration. It does not
need reference/cache databases or the full SED tree. An incremental system such
as Time Machine should protect the live authoritative data and completed SED
products; exclude `backups/` and `seds/*/.work/` if local snapshots and sampler
churn should not be copied off-machine.
