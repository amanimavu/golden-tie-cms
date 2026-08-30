# golden-tie-cms

Headless CMS for the Golden Tie project, built on
[Directus](https://directus.io) 12 (blank starter). Runs entirely in Docker:
Directus + PostgreSQL (PostGIS) + Redis.

## Stack

| Service | Image | Purpose |
| --- | --- | --- |
| `directus` | `directus/directus:12.2.0` | CMS / API / Studio, container port `8055` |
| `database` | `postgis/postgis:16-master` | Primary datastore, port `5432` |
| `cache` | `redis:6` | Cache + rate-limit + WebSocket state |

Defined in [directus/docker-compose.yaml](directus/docker-compose.yaml).

## Prerequisites

- Docker + Docker Compose v2
- Ports free on the host: `8055` (Directus), `5432` (Postgres)

## Quick start

```bash
cd directus
cp .env.example .env        # then edit secrets / config
docker compose up -d
```

First launch: open http://localhost:8055 and complete the onboarding screen to
create the admin account.

Stop / reset:

```bash
docker compose down          # stop, keep data
docker compose down -v       # stop and wipe volumes (Postgres data is a bind
                             # mount at directus/data/database, remove manually)
```

## Configuration

All runtime config is environment-driven. Start from
[directus/.env.example](directus/.env.example); every key is passed through to
the container in `docker-compose.yaml`. Key groups:

- **Database** — `DB_USER`, `DB_PASSWORD`, `DB_DATABASE`
- **Directus** — `DIRECTUS_PORT`, `DIRECTUS_SECRET`, `LICENSE_KEY` (optional)
- **URLs / CORS** — `PUBLIC_URL`, `CORS_ENABLED`, `CORS_ORIGIN`
- **Cookies** — `*_COOKIE_SECURE`, `*_COOKIE_SAME_SITE`, `*_COOKIE_DOMAIN`
- **Cache** — `CACHE_ENABLED`, `CACHE_AUTO_PURGE`
- **WebSockets** — `WEBSOCKETS_ENABLED`
- **Email** — `EMAIL_TRANSPORT`, `EMAIL_SMTP_*`
- **CSP** — `CONTENT_SECURITY_POLICY_DIRECTIVES__FRAME_SRC`

`.env` is gitignored; `.env.example` is the tracked template.

## Layout

```
.
├── README.md                     this file
├── CHANGELOG.md                  upstream starter changelog
├── package.json                  directus:template metadata (Blank starter)
└── directus/
    ├── docker-compose.yaml       services
    ├── .env / .env.example       runtime config
    ├── data/database/            Postgres bind mount (gitignored)
    ├── uploads/                  file storage bind mount (gitignored)
    ├── extensions/               custom extensions bind mount
    └── deploy/nginx/             reverse-proxy virtual host
        ├── README.md             vhost setup guide
        ├── goldentiecms.local.conf   server block (see its README)
        └── map-upgrade.conf      WebSocket Connection-header map
```

## Reverse proxy / virtual host

To serve the Studio on a hostname instead of `localhost:8055` (nginx,
`http://goldentiecms.local`), follow
[directus/deploy/nginx/README.md](directus/deploy/nginx/README.md). It covers the
nginx config, the required `directus/.env` changes, `/etc/hosts`, apply/verify
steps, and the path to HTTPS.

## Common commands

```bash
# tail Directus logs
docker compose -f directus/docker-compose.yaml logs -f directus

# restart just Directus (e.g. after an .env change)
docker compose -f directus/docker-compose.yaml up -d directus

# open a psql shell
docker compose -f directus/docker-compose.yaml exec database \
  psql -U directus -d directus

# health check
curl -sf http://localhost:8055/server/health
```
