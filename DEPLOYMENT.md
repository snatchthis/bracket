# Self-hosting on a home-lab (Docker + Traefik)

Runbook for deploying this fork (`snatchthis/bracket`) on a home-lab using the
prebuilt **combined** image from GitHub Container Registry, behind a Traefik
reverse proxy with HTTPS.

> This is the source-of-truth for an unattended/Claude-Code-driven deploy.
> Files involved: [`docker-compose.prod.yml`](docker-compose.prod.yml),
> [`.env.example`](.env.example), [`Dockerfile`](Dockerfile).

## Architecture

A single **combined** container runs the FastAPI backend which also serves the
prebuilt frontend (`SERVE_FRONTEND=true`). Frontend and API share one origin;
the API lives under `/api`. A separate Postgres container holds the data.
Traefik terminates TLS and routes the public domain to the app on port `8400`.

```
internet ──▶ Traefik (TLS) ──proxy net──▶ bracket:8400 ──bracket_lan──▶ postgres:5432
```

The frontend's API URL is **baked at build time**. The image is built with the
relative path `/api` (see the `VITE_API_BASE_URL` ARG in the Dockerfile), so it
works from any host/domain without rebuilding. Do **not** revert this to an
absolute `http://localhost:...` URL for the combined image.

## Prerequisites on the home-lab

- Docker + Docker Compose v2.
- A running Traefik instance attached to an external Docker network (default
  name assumed here: `proxy`). With entrypoint `websecure` and a TLS
  certresolver configured (default name assumed: `letsencrypt`).
- DNS for your domain pointing at the home-lab (or local DNS / split-horizon).

## Step 1 — Publish the image to ghcr (once per release)

The image must be built **after** the Dockerfile change that sets
`VITE_API_BASE_URL=/api`, otherwise the frontend calls `localhost`.

```bash
# From a clone of this repo (not necessarily the home-lab):
git tag v0.1.1
git push origin v0.1.1   # triggers .github/workflows/docker-publish.yml
```

This publishes `ghcr.io/snatchthis/bracket:v0.1.1` (and `:latest`).
Alternatively run the "docker publish" workflow manually via `workflow_dispatch`.

**Make the package pullable from the home-lab:**
- Either set the ghcr package visibility to **public**, or
- on the home-lab run `docker login ghcr.io -u <user>` with a PAT that has
  `read:packages`.

## Step 2 — Configure on the home-lab

Copy `docker-compose.prod.yml` and create `.env` next to it:

```bash
cp .env.example .env
```

Fill `.env`:

```bash
openssl rand -hex 32     # → JWT_SECRET
openssl rand -hex 24     # → POSTGRES_PASSWORD
```

Required values:

| Variable | Notes |
|----------|-------|
| `BRACKET_DOMAIN` | Public host, e.g. `bracket.example.com`. Must match the Traefik router rule. |
| `BRACKET_TAG` | Image tag to pull, e.g. `v0.1.1` (prefer a pinned tag over `latest`). |
| `JWT_SECRET` | 32-byte hex. **Required in PRODUCTION** — no default. |
| `POSTGRES_PASSWORD` | Strong password. |
| `ADMIN_EMAIL` / `ADMIN_PASSWORD` | Initial admin account, created on first start. |
| `TRAEFIK_CERTRESOLVER` | Name of your Traefik certresolver (default `letsencrypt`). |
| `ALLOW_USER_REGISTRATION` | Keep `false` for a private instance. |

If your Traefik network is **not** named `proxy`, edit it in two places in
`docker-compose.prod.yml`: the `networks.proxy` block and the
`traefik.docker.network` label. Create it if missing: `docker network create proxy`.

## Step 3 — Deploy

```bash
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml logs -f bracket
```

DB migrations and the admin account are created automatically on first start
(`auto_run_migrations` defaults to true). The app is then reachable at
`https://$BRACKET_DOMAIN`.

## Verify

```bash
# Liveness (inside the docker network / via the proxy):
curl -fsS https://$BRACKET_DOMAIN/api/ping        # → "pong"
docker compose -f docker-compose.prod.yml ps      # both services "healthy"
```

Log in with `ADMIN_EMAIL` / `ADMIN_PASSWORD`.

## Upgrades

```bash
# bump BRACKET_TAG in .env (or use a new published tag), then:
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

## Backup / restore (Postgres)

```bash
# Backup
docker compose -f docker-compose.prod.yml exec -T postgres \
  pg_dump -U bracket bracket > bracket-$(date +%F).sql

# Restore (into a fresh, empty DB)
cat bracket-YYYY-MM-DD.sql | docker compose -f docker-compose.prod.yml exec -T postgres \
  psql -U bracket bracket
```

Persistent data lives in the named volumes `bracket_pg_data` (database) and
`bracket_static_data` (uploads).

## Troubleshooting

- **Frontend loads but every request fails / calls `localhost`** — the image was
  built with the old absolute `VITE_API_BASE_URL`. Re-publish from a commit that
  includes the Dockerfile `/api` change and re-pull.
- **Container exits with a config error mentioning `jwt_secret`** — `JWT_SECRET`
  is missing in `.env`; it is mandatory in `PRODUCTION`.
- **502 from Traefik** — wrong `traefik.docker.network`, app not on the `proxy`
  network, or `loadbalancer.server.port` ≠ `8400`.
- **Cannot pull image** — package is private; make it public or `docker login ghcr.io`.
