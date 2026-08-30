# nginx virtual host for Directus

Reverse-proxy the Dockerised Directus instance behind an nginx virtual host on
`http://goldentiecms.local` (plain HTTP, no TLS).

```
browser ──> nginx :80 (goldentiecms.local) ──> 127.0.0.1:8055 ──> directus container :8055
```

The Directus container publishes `${DIRECTUS_PORT}:8055` (default `8055`) on the
host, see `directus/docker-compose.yaml`. nginx forwards to that port on
`127.0.0.1`.

## Files

| File                      | Purpose                                                                           | Deployed to                                                 |
| ------------------------- | --------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `goldentiecms.local.conf` | `server {}` block for the vhost                                                   | `/etc/nginx/sites-available/` → `/etc/nginx/sites-enabled/` |
| `map-upgrade.conf`        | `map` that drives the WebSocket `Connection` header; must load in `http {}` scope | `/etc/nginx/conf.d/`                                        |

Both are symlinked from this directory so the repo stays the source of truth.

## Prerequisites

- nginx installed (`sudo apt install nginx`).
- Directus running: `docker compose -f directus/docker-compose.yaml up -d`.
- This repo checked out at a stable path. Symlinks below point at
  `/home/amani-mavu/Projects/work/golden-tie-cms/...`; adjust if the checkout
  lives elsewhere, and do not move the repo afterwards (a moved repo = dead
  symlink = nginx fails to start).

## 1. nginx server block

`goldentiecms.local.conf`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name goldentiecms.local;

    # Directus file uploads
    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:8055;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;

        # WebSockets (WEBSOCKETS_ENABLED=true); $connection_upgrade comes
        # from map-upgrade.conf
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_read_timeout 300s;
        proxy_buffering off;
    }
}
```

`IP_TRUST_PROXY: "true"` is already set in `docker-compose.yaml`, so Directus
honours the `X-Forwarded-*` headers above.

Enable it (Debian/Ubuntu two-hop convention):

```bash
sudo ln -s \
  /home/amani-mavu/Projects/work/golden-tie-cms/directus/deploy/nginx/goldentiecms.local.conf \
  /etc/nginx/sites-available/goldentiecms.local.conf

sudo ln -s \
  /etc/nginx/sites-available/goldentiecms.local.conf \
  /etc/nginx/sites-enabled/goldentiecms.local.conf
```

## 2. WebSocket map

`map` is only valid in nginx's `http {}` context, not inside a `server {}` or
`location {}` block. On Debian/Ubuntu, `/etc/nginx/conf.d/*.conf` is included at
`http {}` scope, so `map-upgrade.conf` goes there:

```bash
sudo ln -s \
  /home/amani-mavu/Projects/work/golden-tie-cms/directus/deploy/nginx/map-upgrade.conf \
  /etc/nginx/conf.d/map-upgrade.conf
```

It sets `$connection_upgrade` to `upgrade` when the request carries an `Upgrade`
header (WebSocket handshake) and `close` otherwise.

If this vhost will only ever serve Directus, you can skip this file and hardcode
`proxy_set_header Connection "upgrade";` in the `location` block instead.

## 3. Hosts entry

`goldentiecms.local` is not real DNS. Add it on every machine that needs to
reach the Studio:

```bash
echo "127.0.0.1 goldentiecms.local" | sudo tee -a /etc/hosts
```

Note: `.local` is reserved for mDNS/Bonjour. `/etc/hosts` still wins, but on
hosts running Avahi you may see slow lookups. `.test` or `.localhost` avoids
that if you are not tied to `.local`.

## 4. Directus environment

Edit `directus/.env` so Directus generates correct URLs and scopes cookies to
the new host:

| Key                           | From                    | To                          |
| ----------------------------- | ----------------------- | --------------------------- |
| `PUBLIC_URL`                  | `http://localhost:8055` | `http://goldentiecms.local` |
| `CORS_ORIGIN`                 | `*`                     | `http://goldentiecms.local` |
| `REFRESH_TOKEN_COOKIE_DOMAIN` | `localhost`             | `goldentiecms.local`        |
| `SESSION_COOKIE_DOMAIN`       | `localhost`             | `goldentiecms.local`        |

Leave unchanged (plain HTTP, no TLS):

- `REFRESH_TOKEN_COOKIE_SECURE=false`
- `SESSION_COOKIE_SECURE=false`
- `REFRESH_TOKEN_COOKIE_SAME_SITE=lax`
- `SESSION_COOKIE_SAME_SITE=lax`

Optional: if you preview front-end pages inside the Studio, add
`http://goldentiecms.local` to
`CONTENT_SECURITY_POLICY_DIRECTIVES__FRAME_SRC`.

## 5. Apply

```bash
# validate + reload nginx
sudo nginx -t
sudo systemctl reload nginx

# recreate the Directus container with the new env
docker compose -f directus/docker-compose.yaml up -d
```

## 6. Verify

```bash
curl -I http://goldentiecms.local/server/health
# expect HTTP/1.1 200 OK
```

Then open `http://goldentiecms.local` in a browser and confirm the Studio loads,
login works (cookie is set on `goldentiecms.local`), and no CORS errors appear
in the console. Live preview / collaborative editing exercises the WebSocket
path.

## Disable

```bash
sudo rm /etc/nginx/sites-enabled/goldentiecms.local.conf
sudo nginx -t && sudo systemctl reload nginx
```

`sites-available`, `conf.d/map-upgrade.conf`, the repo files, and
`directus/.env` are left in place. Revert the `.env` table above and
`docker compose up -d` again to fully return to `localhost:8055`.

## Moving to HTTPS later

1. Get a cert (mkcert for local, or certbot for a real domain).
2. Add a `listen 443 ssl;` block with `ssl_certificate` / `ssl_certificate_key`,
   redirect `:80` → `:443`.
3. In `directus/.env` set `PUBLIC_URL=https://...`, flip both
   `*_COOKIE_SECURE` to `true`, and set `*_COOKIE_SAME_SITE` as needed.
4. `docker compose -f directus/docker-compose.yaml up -d`.
