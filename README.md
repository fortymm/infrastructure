# fortymm infrastructure

Self-hosted deployment for [fortymm](https://github.com/fortymm). One `docker compose up -d` brings up the database, API, web client, and an nginx reverse proxy that fronts the whole stack on a single port.

## What's in the stack

| Service | Image | Role |
| --- | --- | --- |
| `nginx` | `nginx:alpine` | Reverse proxy. The only service that publishes a host port. Routes `/api/*` to the API (prefix stripped) and everything else to the web client. |
| `web` | `ghcr.io/fortymm/web-client` | Vite SPA served by its own internal nginx. |
| `api` | `ghcr.io/fortymm/api` | FastAPI app, routes mounted under `/v1`. |
| `db` | `postgres:16` | Postgres, data persisted in a named volume. |

The proxy listens on `127.0.0.1:4000` (loopback only). The expectation is that another reverse proxy or tunnel — Cloudflare Tunnel, Caddy, etc. — sits in front of the host and terminates TLS. If you want it reachable from elsewhere on the LAN, change the port mapping in `docker-compose.yml`.

## Requirements

- Docker Engine 24+ with the Compose plugin (`docker compose ...`, not the old `docker-compose`).
- Linux or macOS host. The `api` image is `linux/amd64`, so on Apple Silicon it runs under emulation — fine for staging, slow for hot loops.

## First-time setup

```sh
git clone git@github.com:fortymm/infrastructure.git
cd infrastructure
cp .env.example .env
# edit .env and set strong values for POSTGRES_PASSWORD and JWT_SECRET
docker compose pull
docker compose up -d
```

Once the stack is up, open <http://127.0.0.1:4000>.

## Routing

`nginx/default.conf` defines two locations:

- `/api/*` → `http://api:8000/` with the `/api/` prefix stripped. So a browser request to `/api/v1/ping` hits the api container as `GET /v1/ping`. The strip is done with the trailing slash on `proxy_pass`.
- `/*` → `http://web:80`. The web container serves its own SPA fallback (`try_files ... /index.html`).

Shared proxy headers (`Host`, `X-Forwarded-*`, HTTP/1.1 keepalive) live in `nginx/snippets/proxy.conf` and are included from both location blocks.

If you change the nginx config, reload it without recreating the container:

```sh
docker compose exec nginx nginx -s reload
```

## Web client / API contract

The web client is a Vite SPA, which means `VITE_API_URL` is baked into the bundle at build time, not read at runtime. For this stack to work end-to-end, the published `web-client` image must be built with `VITE_API_URL=/api` so its bundle calls `/api/v1/...`. If you build your own image, pass it as a build arg.

## Updating images

Image tags are pinned to specific SHAs in `docker-compose.yml`. To deploy a newer build:

1. Edit the `image:` line for `web` or `api`.
2. `docker compose pull <service>`
3. `docker compose up -d <service>`

If you redeploy a backend without restarting nginx and start seeing 502s, `docker compose restart nginx` — the embedded nginx caches upstream IPs at config load.

## Common operations

```sh
docker compose ps                      # who's up
docker compose logs -f api             # follow one service
docker compose logs -f                 # follow everything
docker compose restart api             # restart one service
docker compose down                    # stop the stack (data persists)
docker compose down -v                 # stop AND wipe the postgres volume
```

The container names (`fortymm-nginx`, `fortymm-api`, etc.) are pinned, so `docker logs fortymm-api` works directly without compose.

## Backups

Postgres data lives in the `fortymm-db-data` named volume. To dump:

```sh
docker compose exec -T db pg_dump -U fortymm fortymm > backup.sql
```

To restore into a fresh stack:

```sh
docker compose exec -T db psql -U fortymm -d fortymm < backup.sql
```
