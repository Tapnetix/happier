# Self-hosting Happier (single box, Docker)

This guide deploys a complete Happier server on one machine (e.g. Kubuntu) using
the **light flavor**: a single container backed by SQLite and local file storage,
serving both the API and the web UI. **No Postgres, Redis, or S3 required.**

Everything your devices sync — session transcripts, metadata, machine state — is
**end-to-end encrypted** and stored only on your box. See
[`docs/encryption.md`](./encryption.md).

> **Trust boundary.** Self-hosting moves the *sync/relay server* to your machine.
> Happier is still a *client* for coding agents: when an agent runs, it calls its
> own model provider (Anthropic, OpenAI, …) with credentials stored locally on the
> machine running the CLI. Those credentials are encrypted client-side before any
> sync and never reach the server. If you also want zero traffic to model
> providers, point the agent at a local model backend.

## Files in this repo

| File | Purpose |
|---|---|
| `docker-compose.selfhost.yml` | The stack: `happier` server + `caddy` TLS proxy |
| `deploy/selfhost/env.example` | Config template → copy to `.env.selfhost` |
| `deploy/selfhost/Caddyfile` | Reverse-proxy config (auto-HTTPS, WebSocket) |

## Prerequisites

- A host with Docker + Docker Compose v2 (`sudo apt install docker.io docker-compose-v2`).
- A DNS record for your domain (this guide uses `hdev.tapnetix.com`) pointing at
  the host's public IP.
- Ports **80** and **443** reachable from the internet (Caddy needs them for
  Let's Encrypt). The server's `3005` and metrics stay internal — never exposed.

## Setup

```bash
# from the repo root
cp deploy/selfhost/env.example .env.selfhost
chmod 600 .env.selfhost

# generate a master secret and paste it into HANDY_MASTER_SECRET in .env.selfhost
openssl rand -base64 48
```

Edit `.env.selfhost`:
- `HAPPIER_DOMAIN=hdev.tapnetix.com`
- `HANDY_MASTER_SECRET=<the openssl output>` — **back this up off-box.** Losing it
  invalidates existing auth tokens.
- `HAPPIER_PUBLIC_SERVER_URL` / `HAPPIER_WEBAPP_URL` = `https://hdev.tapnetix.com`
- Leave `AUTH_ANONYMOUS_SIGNUP_ENABLED=true` for now (you'll lock it after enrolling).

Bring it up:

```bash
docker compose --env-file .env.selfhost -f docker-compose.selfhost.yml up -d
docker compose --env-file .env.selfhost -f docker-compose.selfhost.yml logs -f happier
```

On first start the server runs its SQLite migrations and Caddy obtains a
certificate. Once healthy, open `https://hdev.tapnetix.com` — you should get the
Happier web UI.

## First run: register, then lock down signups

1. Open `https://hdev.tapnetix.com` in a browser and create your account /
   enroll your devices.
2. Edit `.env.selfhost` → `AUTH_ANONYMOUS_SIGNUP_ENABLED=false`.
3. Re-apply so a stranger can't register against your server:

   ```bash
   docker compose --env-file .env.selfhost -f docker-compose.selfhost.yml up -d
   ```

For stricter access (e.g. GitHub-org allowlist) see the `AUTH_GITHUB_*` variables
in [`docs/deployment.md`](./deployment.md).

## Point the CLI at your server

The CLI defaults to `https://api.happier.dev`. Override it (defaults live in
`apps/cli/src/configuration.ts`):

```bash
export HAPPIER_SERVER_URL=https://hdev.tapnetix.com
export HAPPIER_WEBAPP_URL=https://hdev.tapnetix.com

happier auth login     # links this machine to your self-hosted server
happier                # runs claude via your own relay
```

The CLI persists this as a server profile under `~/.happier/servers/`, so you set
it once per machine.

## Using it from your phone

The **App Store / Play Store apps are compiled against `api.happier.dev` and have
no runtime "custom server" setting** (the URL is baked in at build time via
`EXPO_PUBLIC_HAPPIER_SERVER_URL`). Two options:

- **Recommended — web UI as a PWA.** Open `https://hdev.tapnetix.com` in your
  phone's browser and "Add to Home Screen". The light server serves the full web
  client from your own origin. No rebuild, works immediately.
- **Native app.** Rebuild `apps/ui` from this fork with
  `EXPO_PUBLIC_HAPPIER_SERVER_URL=https://hdev.tapnetix.com` and sideload it. Only
  needed if you specifically require the native app.

## Backups

All state is in the `happier-data` volume: the SQLite DB
(`/data/happier-server-light.sqlite`), uploaded files (`/data/files`), and the
master-secret file if you didn't set one explicitly. Back it up:

```bash
docker run --rm -v happier-selfhost_happier-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/happier-data-$(date +%F).tar.gz -C /data .
```

Also keep `.env.selfhost` (contains `HANDY_MASTER_SECRET`) in your secret store.

## Updating

```bash
docker compose --env-file .env.selfhost -f docker-compose.selfhost.yml pull
docker compose --env-file .env.selfhost -f docker-compose.selfhost.yml up -d
```

Migrations run automatically on startup. Pin `HAPPIER_IMAGE` to a digest in
`.env.selfhost` for reproducible deploys.

## Building from this fork instead of the published image

By default the stack pulls `happierdev/relay-server:preview`. To build the server
from this fork's source instead, edit `docker-compose.selfhost.yml`: comment out
the `image:` line under the `happier` service and uncomment the `build:` block
(`context: .`, `target: relay-server`). Then:

```bash
docker compose --env-file .env.selfhost -f docker-compose.selfhost.yml up -d --build
```

## Hardening checklist

- [ ] Only 80/443 are public; `3005` and metrics (`9090`) stay internal.
- [ ] `HANDY_MASTER_SECRET` set explicitly, `.env.selfhost` is `chmod 600` and
      git-ignored, secret backed up off-box.
- [ ] `AUTH_ANONYMOUS_SIGNUP_ENABLED=false` after enrolling your devices.
- [ ] `METRICS_ENABLED=false` (or scrape it only on a private network).
- [ ] End-to-end encryption left on (default). Do **not** enable the plaintext
      storage feature gate unless you specifically need server-readable storage.
- [ ] `happier-data` volume backed up on a schedule.
- [ ] Host firewall (`ufw allow 22,80,443/tcp`) and unattended security updates.

## Scaling beyond one box

The light flavor is single-node. For horizontal scale (Postgres + Redis + S3,
separate API/worker roles) see [`docs/deployment.md`](./deployment.md); the same
`docker-compose.selfhost.yml` can be adapted by switching `HAPPIER_DB_PROVIDER`,
adding `DATABASE_URL`/`REDIS_URL`/S3 vars, and running a `SERVER_ROLE=worker`
replica.
```

