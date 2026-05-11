# lazy-taskchampion maintainer notes

懒猫微服 lpk wrapper for
[`GothenburgBitFactory/taskchampion-sync-server`](https://github.com/GothenburgBitFactory/taskchampion-sync-server)
— the official sync backend for Taskwarrior 3.x+.

This is a **retag-only** wrapper (no `vendor/`, no `patches/`). The
upstream publishes a complete GHCR image; we point at it by sha256
digest and add a lazycat manifest on top. Same shape as
`lazy-pansou-web`. Build/publish/bootstrap logic lives in
[`microlazy-apps/lazycat-ci`](https://github.com/microlazy-apps/lazycat-ci).

## Lazycat appstore identifiers

- **package id**: `cloud.lazycat.app.lazy-taskchampion`
- **app_id**: `5395` (recorded 2026-05-11, first bootstrap)
- **subdomain**: `taskchampion` → `https://taskchampion.<box-domain>`
- **bootstrap workflow**: when re-running `bootstrap-app.yml` to
  resubmit a fix, pass `app_id=5395` so the workflow skips
  `/app/create` (would 500 on duplicate package).
- **upstream image**: `ghcr.io/gothenburgbitfactory/taskchampion-sync-server`
  - **current pin**: `0.7.1` =
    `sha256:7903477ff4857cc7c702377d1c93a3f6703eed2f89e1733400b8599a4d0bb1bc`
    (resolved 2026-05-11)

## Architecture: single-process Rust HTTP + SQLite

Upstream's `Dockerfile-sqlite` produces a tiny Alpine image:
- `/bin/taskchampion-sync-server` — actix-web HTTP, listens 0.0.0.0:8080
- SQLite db file at `$DATA_DIR/sync-server.db` (default `/var/lib/taskchampion-sync-server/data`)
- entrypoint: chown + `su-exec taskchampion`

No frontend, no compiled-in URLs, no external services. Pure REST API
under `/v1/client/...` plus a `GET /` health string.

```
  task / vit client ──https──► lazycat ingress ──► main:8080 (taskchampion-sync-server)
                                                       │
                                                       └─► SQLite at /lzcapp/var/persist
```

## Why retag instead of vendor + patches

Per the `lazycat-lpk-wrapper` skill's "vendor mode is mandatory"
checklist — none of the triggers fire here:

1. UI hard-codes a SaaS URL? — There is **no UI**. API-only server.
2. Compiled-bundle frontend strings? — No frontend.
3. User-visible URL must come from `.S.AppDomain`? — The user-facing
   URL lives in **client config** (`sync.server.url` in `~/.taskrc`),
   never inside the server's binary.
4. Need entrypoint shim / nginx / supervisord? — Upstream's entrypoint
   already does chown + `su-exec`. Nothing to insert.
5. License? — MIT, binary redistribution explicitly allowed.

So retag-only is correct. Treat the upstream GHCR image as the
ground truth and only override config via env / volumes at install.

## Health check

Upstream exposes no `/health` endpoint, but `GET /` returns
`TaskChampion sync server v0.7.x` with HTTP 200 — that's what we
point both `application.health_check.test_url` and the docker-level
`services.main.healthcheck` at.

Alpine upstream lacks `curl`, so the docker healthcheck uses
`wget -q --spider`. The lazycat-side `health_check.test_url` is a
plain HTTP GET so any working web probe will do.

## Deploy params (one, optional)

`CLIENT_ID` — UUID allowlist mode. Upstream's entrypoint:

```sh
if [ -n "${CLIENT_ID}" ]; then
    export CLIENT_ID
    echo "Limiting to client ID ${CLIENT_ID}"
else
    unset CLIENT_ID
fi
```

So an empty string at install means "no allowlist, any client_id may
register." Most users want that — security is provided by
`sync.encryption_secret`, which the server never sees. Power users
who want strict pinning fill in their UUID.

## Persistence

```yaml
binds:
  - /lzcapp/var/persist:/var/lib/taskchampion-sync-server/data
```

Upstream's Dockerfile declares this path as a VOLUME with owner
`taskchampion:users` (uid 1092). The entrypoint runs
`chown -R taskchampion:users $DATA_DIR && chmod -R 700 $DATA_DIR`
on every start, so binding the lazycat host path is safe — perms
get fixed up before the server starts.

## Refreshing the digest on upstream bump

```sh
TOKEN=$(curl -sSL "https://ghcr.io/token?service=ghcr.io&scope=repository:gothenburgbitfactory/taskchampion-sync-server:pull" | jq -r .token)

# pick a tag (or `latest`)
TAG=0.7.2

curl -sSI -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.oci.image.manifest.v1+json" \
  -H "Accept: application/vnd.oci.image.index.v1+json" \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  -H "Accept: application/vnd.docker.distribution.manifest.list.v2+json" \
  "https://ghcr.io/v2/gothenburgbitfactory/taskchampion-sync-server/manifests/${TAG}" \
  | grep -i docker-content-digest
```

Paste the new digest into `docker/Dockerfile` and bump version in the
release tag.

## Local lpk smoke test

```sh
# one-time
git clone https://github.com/microlazy-apps/lazycat-ci ~/lazycat-ci

# in this repo:
bash ~/lazycat-ci/scripts/build-lpk.sh \
  --image lazy-taskchampion:dev \
  --version 0.0.0-dev \
  --docker-context ./docker
# → lazycat/cloud.lazycat.app.lazy-taskchampion-0.0.0-dev.lpk
```

Build is fast since the image is just retagged from GHCR — no Rust
compilation.

## Client setup quick reference (for support questions)

The server has no UI; troubleshooting almost always lives on the
client side (`task` / `vit`). Three `~/.taskrc` keys do everything:

| Key | Purpose | Per-machine? |
|---|---|---|
| `sync.server.url` | base URL of this lpk | same |
| `sync.server.client_id` | UUID identifying *the task dataset* (NOT the machine) | **same** |
| `sync.encryption_secret` | key for end-to-end encryption | **same** |

The server isolates history chains **per client_id**, so replicas
sharing tasks must share the same UUID. Verified by direct API probe
2026-05-11: `GET /v1/client/get-child-version/NIL` with two different
`X-Client-Id` headers returns two independent chains (one 200 + data,
one 404).

vit has no sync code of its own — `S` → `subprocess(['task','sync'])`.
Config to auto-sync on vit start/quit lives in `~/.vit/config.ini`
(`sync_on_start` / `sync_on_end`).

## Versioning notes

- Upstream uses semver `0.x.y` (currently 0.7.1).
- This wrapper picks its own semver, starting at `v0.0.1`.
- Lazycat appstore enforces strict monotonic semver per package id.
