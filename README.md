# photo-gallery-deploy

Deployment configuration for the Photo Gallery project. This repo owns the
single VPS that runs both apps together — it does not build any images, it
only pulls and runs what `photo-gallery-server` and `photo-gallery-client`
publish to GHCR.

Neither app repo should know the other exists. If you find yourself adding
a reference to one app's image inside the other app's repo, it belongs
here instead.

## What's here

- `compose.yaml` — runs `server`, `react-prod`, `caddy`, and `db` together
  on one Docker network.
- `Caddyfile` — reverse proxy + automatic HTTPS (Let's Encrypt) for:
  - `jmediatechnology.com`, `www.jmediatechnology.com`,
    `photo-gallery.jmediatechnology.com` → `react-prod:8080`
  - `api.jmediatechnology.com` → `server:80`
- `secrets/` — gitignored. Populate on the server only (see below).

## One-time server setup

1. Clone this repo onto the VPS, e.g. `/opt/photo-gallery-deploy`.
2. Create the secret files (these are never committed):
   ```bash
   printf '%s' 'your-db-password'      > secrets/db-password-prod.txt
   printf '%s' 'your-anthropic-key'    > secrets/anthropic-api-key.txt
   ```
   Use `printf '%s'`, not `echo` or an editor save — a trailing newline
   breaks `file_get_contents()`-based secret reads on the Symfony side.
3. Confirm DNS A records exist and point at this server's IP:
   - `jmediatechnology.com`
   - `www.jmediatechnology.com`
   - `photo-gallery.jmediatechnology.com`
   - `api.jmediatechnology.com`
4. Confirm the Hetzner Cloud Firewall allows inbound 80 and 443 (not just
   the old 8080/9000 dev ports).

## Deploying / updating

```bash
docker compose pull
docker compose up -d
```

This pulls the `:latest` tags for `photo-gallery-server` and
`photo-gallery-client` from GHCR and restarts anything that changed. Caddy
requests/renews TLS certificates automatically on first request per
hostname — no manual certbot step.

## Verifying

```bash
docker compose ps
docker compose logs -f caddy
```

Watch the Caddy log on first deploy to confirm certificate issuance
succeeds for all four hostnames.

## Known dependencies on the app repos

- `photo-gallery-client`'s `.env.production` must set
  `VITE_API_URL=https://api.jmediatechnology.com` (baked in at image build
  time, not runtime — a change there requires a client image rebuild).
- `photo-gallery-server`'s CORS config (`CORS_ALLOW_ORIGIN` env /
  `config/packages/nelmio_cors.yaml`) must allow the exact origins the
  client is served from, e.g.:
  ```
  ^https://(www\.|photo-gallery\.)?jmediatechnology\.com$
  ```
