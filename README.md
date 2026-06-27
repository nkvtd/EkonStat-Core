# EkonStat-Core

[![Node.js](https://img.shields.io/badge/node-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![PostgreSQL](https://img.shields.io/badge/postgresql-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)

This repository provides a single deployment that ties together [EkonStat-API](https://github.com/nkvtd/EkonStat-API) and [EkonStat-Dashboard](https://github.com/nkvtd/EkonStat-Dashboard). With a configuration file and a database password, the entire stack comes up — no local build step, no source code to mount.

## What's in the stack

**EkonStat-API** is a tool that consolidates public procurement data for Macedonia. It provides a modular API layer with shared infrastructure and validation, dedicated worker runtimes for scheduled ingestion and one-off backfills, and event-driven webhook fanout when new data arrives. The stack is built around Hono, Drizzle ORM, PostgreSQL, and worker processes. The live instance runs at [ekonstat.nkvtd.com/api/](https://ekonstat.nkvtd.com/api/).

**EkonStat-Dashboard** is the public-facing frontend — a Vue 3 dashboard for browsing and visualizing that data. It offers searchable, filterable tables with cursor-paginated infinite loading, detail drawers for individual contracts and institutions, and support for three locales (Macedonian, English, Albanian), two display currencies (MKD, EUR), and a dark/light theme with persisted preferences. It's a client-rendered SPA served through Nginx that reverse-proxies `/api` back to the API. The live instance is at [ekonstat.nkvtd.com](https://ekonstat.nkvtd.com).

**Cloudflared Tunnel** provides a secure, outbound-only connection from the server to Cloudflare's edge — no inbound firewall ports need to be opened. It authenticates via a tunnel token and sits on the shared `edge-proxy` Docker network alongside the reverse proxies.

**Watchtower** quietly polls the container registries every five minutes. When a new image is published for any opted-in container, it pulls it down, gracefully restarts that container, cleans up the old image, and fires off a notification through Shoutrrr to any compatible service — Discord, Slack, Telegram, or email. Only containers explicitly tagged with `com.centurylinklabs.watchtower.enable=true` are monitored, so nothing updates unless it has been explicitly opted in.

## Architecture

```
                    Internet
                       |
              Cloudflare Tunnel                     (tunnel profile)
                       |
             edge-proxy (external network)
                  /         \
         dashboard (Nginx)   app (Hono API, :8080)
         /api ------> app:8080    |
                                  |
                           postgres :5432
                                  |
                           scheduler (cron, webhooks)
                           backfiller              (backfill profile)
                           migrate                 (tools profile)

              Watchtower                           (watchtower profile)
```

## Quick Start

The only prerequisites are Docker and Docker Compose. Everything else is handled by the compose file.

**1. Set up the configuration**

```bash
git clone https://github.com/nkvtd/EkonStat-Core.git && cd EkonStat-Core
cp .env.example .env
```

Open `.env` and fill in the required values. If the tunnel or Watchtower profiles are needed, set `TUNNEL_TOKEN` and `WATCHTOWER_NOTIFICATION_URL` as well.

**2. Generate the database password**

```bash
mkdir -p secrets
openssl rand -base64 32 > secrets/database_password.txt
```

This becomes a Docker secret — the password never touches `.env`.

**3. Create the edge network**

```bash
docker network create edge-proxy
```

The `dashboard`, `app`, and `tunnel` services all attach to this external network so Cloudflare can reach them.

**4. Start Postgres and run migrations**

```bash
docker compose up -d postgres
docker compose --profile tools run --rm migrate
```

The migrate container uses the same API image, reads the secret to construct its `DATABASE_URL`, and applies the Drizzle migration SQL against the database. It runs once and exits.

**5. Bring up the core stack**

```bash
docker compose up -d app scheduler dashboard
```

**6. Check that everything is healthy**

```bash
docker compose exec app wget -q -O - http://localhost:8080/api/ping
docker compose exec app wget -q -O - http://localhost:8080/api/ready
```

### Adding optional services later

The tunnel, Watchtower, and backfiller are gated behind profiles so they don't start unless explicitly enabled:

```bash
# Continuous ingress through Cloudflare
docker compose --profile tunnel up -d

# Automatic container image updates
docker compose --profile watchtower up -d

# One-off historical backfill — runs and exits
docker compose --profile backfill up backfiller
```

Profiles can be stacked: `docker compose --profile tunnel --profile watchtower up -d`.

## Services

| Service | Image | Profile | What it does |
|---------|-------|---------|--------------|
| `dashboard` | `ghcr.io/nkvtd/ekonstat-dashboard:latest` | default | Vue 3 frontend served through Nginx. The browser makes same-origin `/api` requests, Nginx catches them and proxies to `API_UPSTREAM`. |
| `postgres` | `postgres:18-alpine` | default | PostgreSQL 18 database. Port `5432` is bound to `127.0.0.1` on the host. Data lives in a named volume at `/var/lib/postgresql`. |
| `app` | `ghcr.io/nkvtd/ekonstat-api:latest` | default | The Hono API server, listening on `:8080` internally. Exposes `/api/ping`, `/api/ready`, and the full contract data API. Healthchecked via `wget` every 30 seconds. |
| `scheduler` | `ghcr.io/nkvtd/ekonstat-api:latest` | default | A cron-based worker that ingests procurement data on a schedule and dispatches webhook events. It reads webhook URLs from `.env` via `env_file`. |
| `backfiller` | `ghcr.io/nkvtd/ekonstat-api:latest` | `backfill` | Runs the same ingestion strategies as the scheduler, but in bulk backfill mode for historical data. Runs once and exits (`restart: "no"`). |
| `tunnel` | `cloudflare/cloudflared:latest` | `tunnel` | Cloudflare Tunnel daemon. Creates an outbound-only connection to Cloudflare's edge — no inbound ports needed. Attached to the `edge-proxy` network. |
| `watchtower` | `nickfedor/watchtower:1.13.1` | `watchtower` | Polls image registries every 5 minutes for new versions of opted-in containers. Pulls, restarts, cleans up old images, and sends notifications through Shoutrrr. |
| `migrate` | `ghcr.io/nkvtd/ekonstat-api:latest` | `tools` | A one-shot container that constructs `DATABASE_URL` from the Docker secret, runs `npm run db:migrate`, and exits. |

## Environment Variables

All variables live in `.env`. Copy `.env.example` and fill in the values.

| Variable | Default | Required | What it controls |
|----------|---------|----------|-----------------|
| `POSTGRES_USER` | `postgres` | No | PostgreSQL user for the `ekonstat-db` database. |
| `POSTGRES_DB` | `ekonstat-db` | No | Database name. |
| `PORT` | `8080` | No | The port the API server listens on inside its container. |
| `API_UPSTREAM` | `http://app:8080` | **Yes** | The backend URL the dashboard's Nginx proxies `/api` requests to. Must be reachable from the dashboard container on the Docker network. |
| `TUNNEL_TOKEN` | — | If using `tunnel` | A Cloudflare Tunnel token, created in the Cloudflare Zero Trust dashboard. |
| `WATCHTOWER_NOTIFICATION_URL` | `discord://token@channelid` | No | A [Shoutrrr](https://containrrr.dev/shoutrrr/) URL that tells Watchtower where to send update notifications. |
| `REALISED_CONTRACTS_WEBHOOK_N` | — | No | Webhook URLs called when new realised contracts are discovered. Numbered suffixes (`_1`, `_2`, ... `_N`) allow multiple endpoints to be registered. |
| `AWARDED_CONTRACTS_WEBHOOK_N` | — | No | Same pattern for newly discovered awarded contracts. |
| `CHANGES_IN_AWARDED_CONTRACTS_WEBHOOK_N` | — | No | Same pattern for contract change events. |

If no webhook variables with a given prefix exist, dispatch is skipped for that event type. The webhook payload includes a `data` array and a `meta` object with the event name and dispatch timestamp.

## How things work

### The database password stays out of `.env`

The password lives in `./secrets/database_password.txt` and is mounted as a Docker secret at `/run/secrets/database_password` inside every container that needs it — `postgres`, `app`, `scheduler`, `backfiller`, and `migrate`. At startup, the API-derived containers read the secret file, construct the full `DATABASE_URL` from `POSTGRES_USER`, `POSTGRES_DB`, and the password, and export it before launching their respective `npm run` commands. The password is never written to `.env` or any environment block.

### The `edge-proxy` network

Declared as `external: true` in the compose file, the network must exist before containers attach. Services that need to be reached from the outside — `dashboard` and `app` — join it alongside the Cloudflare tunnel. Internal-only services like `postgres`, `scheduler`, `backfiller`, and `migrate` communicate over the implicit `default` network.

### Watchtower's opt-in model

Watchtower only monitors containers tagged with `com.centurylinklabs.watchtower.enable=true` — so only `dashboard`, `app`, and `scheduler`. It checks for updates every 300 seconds, cleans up the old image after a successful pull, and fires a per-container notification through Shoutrrr. Set `WATCHTOWER_NOTIFICATION_URL` to any Shoutrrr-compatible service:

```env
WATCHTOWER_NOTIFICATION_URL=discord://token@channelid
# or
WATCHTOWER_NOTIFICATION_URL=slack://tokenA/tokenB/tokenC
# or
WATCHTOWER_NOTIFICATION_URL=telegram://token@telegram?chats=@channel-id
```

### Cloudflare Tunnel

The tunnel creates a secure, outbound-only pipe to Cloudflare's edge — no firewall rules to manage. After [creating a tunnel](https://developers.cloudflare.com/tunnel/advanced/tunnel-tokens/) in the Cloudflare dashboard and setting the token in `.env`, start the profile:

```bash
docker network create edge-proxy
docker compose --profile tunnel up -d
```

## License

This project is licensed under the MIT License.
