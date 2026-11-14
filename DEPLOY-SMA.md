# Cloudreve tester-env deployment (SMA)

Reproducible source-built environment for Cloudreve, used by the RL training
corpus to run nominal UI scenarios with pass/fail mutations.

## Baseline and provenance

| Part | Ref | Notes |
| --- | --- | --- |
| Backend | `cloudreve/Cloudreve` 4.19.1 = `a8becb9f5b226024c83230bc52857c04966e2c30` | Latest stable GitHub Release at triage (2026-09-18). Release tag used; no branch/commit fallback needed. |
| Frontend | `cloudreve/frontend` `19da0fe1ecd40971fafa813983d769fdce41573c` | The `assets` gitlink pinned by the 4.19.1 tag. The frontend repo publishes no tags, so the gitlink is the authoritative pin. |

The frontend is **vendored** into `assets/` instead of being kept as a submodule:
the upstream `.gitmodules` entry was removed and the frontend tree at
`19da0fe1` was copied into `assets/` (its `.git` directory dropped). Reason: a
mutation patch against the parent tree must be able to touch frontend code; a
submodule pointer cannot be patched by the parent scenario patch.

The corpus consumes this tree from our fork
`https://github.com/sma-rl-dev/cloudreve` (branch `tester-env-baseline`) as the
`apps/cloudreve` submodule. The fork baseline starts at upstream release 4.19.1
= `a8becb9f5b226024c83230bc52857c04966e2c30` and adds the whole tester-env
deployment layer in one commit: `Dockerfile.tester-env`,
`docker-compose.tester-env.yml`, `.dockerignore`, `tester-env`, `seed/data/**` +
`seed/timestamps.tsv`, this file, and the vendored `assets/` frontend tree
(replacing the upstream `assets` gitlink and removing `.gitmodules`).
`status/apps/cloudreve.toml` in the corpus pins the immutable baseline commit
SHA of that branch tip. Every CLI path is derived from the script location, so
the same commands work from a fork checkout and from `apps/cloudreve`.

Reproduce the vendoring from a clean checkout:

```bash
git clone https://github.com/cloudreve/Cloudreve.git cloudreve
cd cloudreve && git checkout 4.19.1
git clone https://github.com/cloudreve/frontend.git /tmp/cr-frontend
cd /tmp/cr-frontend && git checkout 19da0fe1ecd40971fafa813983d769fdce41573c
cd - && git rm --cached assets && rm -rf assets .gitmodules && mkdir assets
cp -a /tmp/cr-frontend/. assets/ && rm -rf assets/.git
```

## Quick start

```bash
./tester-env deploy    # build image from source, start container, bootstrap admin, seed data
./tester-env seed      # (re)apply the deterministic seeded baseline (idempotent)
./tester-env verify    # assert version, SPA shell, login, and every seeded record
./tester-env reset     # down -v: container + data volume removed
./tester-env status    # running?, port, which statics source
./tester-env logs      # follow container logs
```

Default local URL `http://localhost:18610/`; browser-container URL
`http://host.docker.internal:18610/`. `--run-id <id>` scopes compose project,
container and volume; `--port <port>` moves the host port. `scripts/rl-env`
supplies `IMAGE_TAG`, `RUN_ID`, `PORT` and `TESTER_ENV_BROWSER_ORIGIN`.

## Deterministic credentials

| Field | Value |
| --- | --- |
| Email | `admin@cloudreve.local` |
| Password | `CloudreveAdmin!2026` |
| Role | admin (group 1, auto-promoted as the first registered user) |

Overridable with `CLOUDREVE_ADMIN_EMAIL` / `CLOUDREVE_ADMIN_PASSWORD`. There is
no install wizard and no user-create CLI; the deployment registers the first
user through the public registration endpoint (open by default: no captcha, no
email activation).

## Build topology (fork-owned `Dockerfile.tester-env`)

Upstream ships no source-build Dockerfile (its Dockerfile only `COPY`s a
prebuilt `cloudreve` binary), so this file replaces it:

1. `frontend` — `node:22-bookworm`, `apt-get install zip`, `yarn install
   --frozen-lockfile` (BuildKit cache mount on the yarn cache, keyed on
   `assets/package.json` + `assets/yarn.lock`), `yarn version --new-version
   4.19.1`, `NODE_OPTIONS=--max-old-space-size=8192 yarn run build`, then
   `zip -qr - assets/build > assets.zip` (members must be `assets/build/...`
   because `application/statics/embed.go` does `fs.Sub(statics, "assets/build")`).
2. `backend` — `golang:1.26`, `go mod download` (module cache mount), then
   `CGO_ENABLED=0 go build -trimpath -buildvcs=false` with
   `-X ...constants.BackendVersion=4.19.1 -X ...constants.LastCommit=a8becb9f`.
   The frontend `assets.zip` is copied to `application/statics/assets.zip` before
   the build; a bare `go build` on a fresh checkout fails without it.
3. `runtime` — `alpine:3.22` + tzdata/ca-certificates, single static binary,
   `WORKDIR /cloudreve`, `VOLUME /cloudreve/data`, `EXPOSE 5212`.

Deliberately **not** installed: vips, ffmpeg, libreoffice, aria2, supervisor from
the upstream image. They only back thumbnail/media-metadata previews and the
aria2 relay; those UI surfaces are out of scope, so do not assert on them.

Measured build/rebuild costs (2026-11-14, 28-core / 62 GB host, cold cache):

| Operation | Time |
| --- | --- |
| Frontend stage (yarn install + vite build) | ~3m36s |
| Backend (go mod download + go build) | ~1m23s |
| Cold full build | ~5 min |
| Backend-only source edit (`service/` change) rebuild | ~3-4 s compile, image digest unchanged for comment-only edits |
| Frontend-only mutation via `./tester-env frontend` | ~1m30s (vite rebuild + restart) |
| `reset` + `deploy` + `verify` (image cached) | ~20 s |

## Runtime configuration

No database, cache, worker or paid service: the defaults are sqlite
(`modernc.org/sqlite`, pure Go) at `data/cloudreve.db`, the in-process
file-backed KV store instead of Redis, and a local storage policy created by the
migration (`data/uploads/{uid}/{path}`, 25 MB chunks).

All mutable state lives under `/cloudreve/data` (`util.DataPath()` =
`<exe dir>/data`): `conf.ini` (auto-created with `[System] Listen = :5212`),
`cloudreve.db`, `uploads/`, `temp/`, `avatar/`, and an optional `statics/`
override. That is the single volume `down -v` drops.

`CR_SETTING_DEFAULT_siteURL` is set from `TESTER_ENV_BROWSER_ORIGIN` (or
`http://localhost:<port>`). It is consumed by `inventory/migration.go` as the
*initial DB default*, so absolute links (share links) point at a host the
browser container can reach, while admin edits of the setting still work.

## Browser evidence

All runs used the browser container origin `http://host.docker.internal:18610/`,
Chromium at 1280x800, login `admin@cloudreve.local / CloudreveAdmin!2026`.

* Smoke (2026-11-14, pre-seed empty baseline): created a folder `Smoke Review`
  in My Files, it appeared in the list and sidebar; deleted it and the trash
  dialog/notification appeared; Trash listed it with original location
  `My Files`; `Empty trash` reported `Trash has been emptied`; the refreshed
  My Files showed `Nothing was found`.
* Seed verification (2026-11-14, freshly reset -> deploy -> seeded baseline):
  My Files showed the 4 root folders `Archive`, `Clients`, `Finance`, `Team`;
  `Clients` showed `Contracts`, `Harbor Health`, `Kestrel Retail`;
  `Kestrel Retail` showed the 3 named files; `Finance/Invoices` showed the 3
  invoice CSVs; `Team/Meeting Notes` showed the 3 named notes; Shares showed
  `Harbor Health` (public folder, no lock) and `kestrel-brief.txt` (lock icon);
  Trash showed `Nothing was found`. The check did not mutate any data.

## Reset

```bash
./tester-env reset    # docker compose down -v --remove-orphans
./tester-env deploy   # fresh sqlite migrations + admin bootstrap + deterministic seed
```

Verified: two consecutive `reset` -> `deploy` -> `verify` cycles from a pristine
volume recreated the same credentials and the identical 30-row seeded snapshot
(11 folders, 17 files with fixed sizes/mtimes, 2 shares) on the host
`http://host.docker.internal:18610/`.

## Mutation paths

* Backend mutation: edit Go source, `./tester-env deploy` (BuildKit reuses the
  Go module/build caches; only changed packages recompile).
* Frontend mutation: edit `assets/src/...`, then `./tester-env frontend`. This
  rebuilds only the vite bundle through the `statics` compose profile and copies
  it into the data volume as `data/statics`, which
  `application/statics/statics.go` prefers over the embedded zip. No Go rebuild.
  `./tester-env deploy` removes `data/statics` again so the image's own bundle
  wins. Proven with a temporary `<title>` literal: the rebuilt string was served
  at `/` after `frontend` and the baseline string returned after `deploy`.

## Deterministic seed (Northwind Studio archive)

`./tester-env seed` (and `./tester-env deploy`, which seeds after the admin
bootstrap) creates a small design-studio document archive through Cloudreve's
authenticated API. Fixture bytes live under `seed/data/` and their fixed
modified times in `seed/timestamps.tsv`; both are committed, so the directory
tree maps 1:1 to `cloudreve://my/`.

| Path | Records |
| --- | --- |
| My Files | 4 folders: `Archive`, `Clients`, `Finance`, `Team` |
| Clients | `Contracts`, `Harbor Health`, `Kestrel Retail` |
| Clients/Kestrel Retail | `kestrel-brief.txt` (442 B), `kestrel-launch-checklist.csv` (247 B), `kestrel-wireframe-notes.txt` (333 B) |
| Clients/Harbor Health | `harbor-brand-palette.csv` (157 B), `harbor-logo-feedback.txt` (255 B) |
| Clients/Contracts | `kestrel-sow-2026.txt` (344 B), `harbor-retainer-2026.txt` (267 B) |
| Finance/Invoices | `invoice-2026-01.csv` (139 B), `invoice-2026-02.csv` (132 B), `invoice-2026-03.csv` (132 B) |
| Finance/Expenses | `travel-q1-2026.csv` (178 B) |
| Team/Meeting Notes | `kickoff-2026-01-08.txt` (251 B), `design-review-2026-02-12.txt` (215 B), `kestrel-retrospective-2026-03-20.txt` (255 B) |
| Team/Onboarding | `studio-handbook.txt` (233 B), `tool-access-checklist.txt` (179 B) |
| Archive | `old-site-copy-2025.txt` (188 B) |

Totals: 11 folders, 17 files, 3,947 bytes of quota used, 2 shares.

Fixed file modified times (list "Modified" column, from
`seed/timestamps.tsv`): the newest is `kestrel-launch-checklist.csv`
(2026-03-19T16:45:00Z) and the oldest `old-site-copy-2025.txt`
(2025-06-30T18:00:00Z); every other file has a pinned date in
2025-06-30..2026-04-02. Folder creation timestamps are install time, so assert
file dates only.

Shares (natural to the archive, visible on the Shares page):

| Source | Type | Protection |
| --- | --- | --- |
| `Clients/Harbor Health` | folder | public link |
| `Clients/Kestrel Retail/kestrel-brief.txt` | file | password `harbor2026` (lock icon) |

`./tester-env verify` is the seed gate. On top of the version/SPA/login probes
it asserts, per directory, the exact child folder names and the exact
`name | bytes | modified epoch` rows for files, then the empty trash and the
two shares (count, names, source types, one password-protected). It fails on
extra, missing, renamed or resized records.

Determinism and idempotence:

* Upload sessions pass `last_modified`, which Cloudreve stores as the file's
  `updated_at`, so modified dates replay exactly after `reset`.
* Re-running `seed` re-creates folders by name (the API returns the existing
  node), skips a file whose size and modified time already match (so no extra
  version or quota is created), re-uploads drifted files as a new version, and
  skips a share when a share for that file name already exists.
* Verified: two `reset` -> `deploy` cycles plus one explicit re-seed produced
  identical 30-row API snapshots and `verify` passed after each.
* Never assert ent hashids, share ids/URLs (host is `siteURL`), storage names
  or folder timestamps.

## API contract used by the seed

```bash
BASE=http://127.0.0.1:18610
# 1. login -> bearer token (JSON: {"code":0,"data":{"user":...,"token":{"access_token":...}}})
TOKEN=$(curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"email":"admin@cloudreve.local","password":"CloudreveAdmin!2026"}' \
  "$BASE/api/v4/session/token" | jq -r '.data.token.access_token')
# 2. create a folder (uri scheme is cloudreve://<filesystem>/<path>; "my" = own files)
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"uri":"cloudreve://my/Projects","type":"folder"}' "$BASE/api/v4/file/create"
# 3. list a directory (response items live in data.files[])
curl -s -G -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'uri=cloudreve://my' --data-urlencode 'page_size=50' "$BASE/api/v4/file" \
  | jq -c '.data.files[]|{name,type,size,updated_at}'
# 4. upload a file: create a session (entity_type=version makes re-seed overwrite),
#    then POST one raw-body chunk
SESS=$(curl -s -X PUT -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"uri":"cloudreve://my/probe.txt","size":23,"last_modified":1768208400000,"mime_type":"text/plain","entity_type":"version"}' \
  "$BASE/api/v4/file/upload" | jq -r '.data.session_id')
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/octet-stream' \
  --data-binary @probe.txt "$BASE/api/v4/file/upload/$SESS/0"
# 5. share a file/folder (is_private=true + password for a protected link)
curl -s -X PUT -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"uri":"cloudreve://my/Projects","is_private":false}' "$BASE/api/v4/share"
```

Other authenticated route groups usable for later seed expansion: `/api/v4/file`
(create/rename/move/delete/restore/lock/metadata PATCH/search), `/api/v4/share`,
`/api/v4/user/me`, `/api/v4/user/capacity`, and the admin console under
`/api/v4/admin/{summary,group,user,policy,share,task,entity,node,oauthClient,settings}`.
Bootstrap-relevant defaults: `register_enabled=1`, `login_captcha=0`,
`reg_captcha=0`, `email_active=0`, `default_group=2`, first user → group 1.

The static seed manifest for scenario seed checks lives at
`scenarios/cloudreve/seed.manifest.json` in the corpus repo.

## Caveats

* `siteID`, session secret, secret key and every ent hashid are per-install
  random; uploaded storage names embed `{uid}_{randomkey8}_{originname}`. Assert
  display names, counts, ordering, permissions and relationships — never ids,
  storage names or absolute timestamps.
* No image preview/thumbnail support (see build topology); skip
  thumbnail/media-metadata assertions.
* The PWA service worker registers only in secure contexts, so the
  `http://host.docker.internal:<port>/` browser path never caches; clear site
  data when testing on a `localhost`-served instance.
* Share links and emails use the DB-seeded `siteURL`; changing it in the admin
  console works and survives restarts (it is a migration default, not a hard
  env override).
* The image build needs network (yarn registry, Go module proxy) and about
  6-8 GB of Docker storage for the build cache; the host home volume runs close
  to full, so avoid parallel Cloudreve builds.
