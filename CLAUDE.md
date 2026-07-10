# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single-file `docker-compose.yml` that stands up a self-hosted [Langfuse](https://langfuse.com) v3 observability stack on a single VM. The application code lives upstream in the `docker.io/langfuse/langfuse:3` and `docker.io/langfuse/langfuse-worker:3` images — this repo only owns the deployment topology, secrets, and infrastructure defaults.

## Architecture

Six services wired together. Inbound traffic only reaches the two front-door services; everything else binds to `127.0.0.1`.

| Service           | Image                                       | Purpose                                                | Host port |
|-------------------|---------------------------------------------|--------------------------------------------------------|-----------|
| `langfuse-web`    | `docker.io/langfuse/langfuse:3`             | Next.js UI + ingestion API (the user-facing app)      | `3000`    |
| `langfuse-worker` | `docker.io/langfuse/langfuse-worker:3`      | Async processor: writes events to ClickHouse, S3       | _network-only_ |
| `postgres`        | `postgres:${POSTGRES_VERSION:-17}`          | Auth, orgs, projects, users, API keys (relational)     | _network-only_ |
| `clickhouse`      | `clickhouse/clickhouse-server`              | Columnar store for traces, observations, scores       | _network-only_ |
| `minio`           | `cgr.dev/chainguard/minio`                  | S3-compatible blob store for events, media, exports    | `9090` (API only) |
| `redis`           | `redis:7`                                   | BullMQ job queue, ingestion buffers                    | _network-only_ |

**Request flow for a typical LLM trace:** SDK → `langfuse-web` POST → enqueue on Redis → `langfuse-worker` pops job → writes observation rows to ClickHouse and any blob payload to MinIO → metadata (project, prompt, score) stays in Postgres.

**State lives in named Docker volumes** (`langfuse_postgres_data`, `langfuse_clickhouse_data`, `langfuse_clickhouse_logs`, `langfuse_minio_data`, `langfuse_redis_data`). The repo contains no application code — those volumes ARE the data.

## Secrets & config

All config is read from a single `.env` file. Every variable referenced by `docker-compose.yml` is listed in `.env.example` with sensible local-dev defaults. The compose file falls back to the same defaults if `.env` is missing. Rotate secrets for production with `openssl rand -hex 32`.

Replace these secrets before exposing anything beyond localhost:

- `POSTGRES_PASSWORD` / `DATABASE_URL` (both must agree)
- `SALT` and `ENCRYPTION_KEY` — generate `ENCRYPTION_KEY` with `openssl rand -hex 32` (must be 64 hex chars). Once set, never change it without re-encrypting stored data.
- `NEXTAUTH_SECRET`
- `CLICKHOUSE_PASSWORD`
- `REDIS_AUTH` (must match the `command:` block on the `redis` service)
- `MINIO_ROOT_PASSWORD` and the three `LANGFUSE_S3_*_SECRET_ACCESS_KEY` values

Optional bootstrap (otherwise create org/project/user from the web UI on first load): `LANGFUSE_INIT_ORG_*`, `LANGFUSE_INIT_PROJECT_*`, `LANGFUSE_INIT_USER_*`.

## Commands

All commands assume CWD is the repo root. The `.env` file is picked up automatically by Compose.

```bash
# Start the stack (foreground, Ctrl-C to stop)
docker compose up

# Start detached
docker compose up -d

# Tail logs for the app only
docker compose logs -f langfuse-web langfuse-worker

# Check that every service is healthy
docker compose ps

# Stop containers, keep volumes
docker compose down

# Nuke everything including all data volumes — IRREVERSIBLE
docker compose down -v

# Upgrade to the latest v3 image tag
docker compose pull && docker compose up -d

# Restart a single service after a config change
docker compose restart langfuse-web

# Open a psql / clickhouse-client / redis-cli shell
docker compose exec postgres psql -U postgres
docker compose exec clickhouse clickhouse-client
docker compose exec redis redis-cli -a "$REDIS_AUTH"
```

There is no build, lint, or unit-test step in this repo — the application code is upstream. The equivalent health checks are `docker compose ps` (all services `healthy`) and a `200 OK` from `http://localhost:3000/api/public/health`.

## Backups

The compose stack has no built-in backup; you'll need to script against the running containers. Postgres and ClickHouse hold the data you cannot reconstruct; MinIO holds blobs that are usually regenerable.

```bash
# Postgres logical dump
docker compose exec -T postgres pg_dump -U postgres -Fc postgres > backups/postgres-$(date +%F).dump

# ClickHouse snapshot
docker compose exec -T clickhouse clickhouse-backup create "$(date +%F)" || true

# MinIO bucket mirror (needs the mc client)
docker run --rm -it --network langfuse-stack_default \
  -v "$PWD/backups:/backups" minio/mc \
  mirror --overwrite http://minio:9000/langfuse /backups/langfuse
```

Restore by `docker compose exec -T postgres pg_restore` and re-pushing MinIO objects. For prod, use the Kubernetes/Helm deployment — Compose is intentionally single-node and stateless against node loss.

## Things that bite people

- **MinIO is not reachable from outside the compose network for direct uploads.** Multimodal traces that send media to `http://minio:9000` from a host-side SDK will fail. Either set `LANGFUSE_S3_MEDIA_UPLOAD_ENDPOINT` to `http://localhost:9090` (default) and let the SDK upload through the web proxy, or front MinIO with a reverse proxy. See the Langfuse blob-storage docs.
- **Rotating `ENCRYPTION_KEY` corrupts all stored API keys.** Rotate it by adding a second key and re-encrypting, never by direct replacement.
- **`LANGFUSE_INIT_*` only runs on a fresh database.** Re-setting them after first boot has no effect — manage users/projects through the UI or API.
- **Port 3000 must be unique on the host** — if a previous Langfuse container holds it, `docker compose up` will silently fail to bind. Run `docker compose down` on the old stack first.
- **Health check takes 30–60s on cold start** because ClickHouse and Postgres migrate before `langfuse-web` reports healthy. Don't restart-loop on it.

## Reference

- Upstream compose file: `docker-compose.yml` (vendored from `langfuse/langfuse@main`). When upgrading Langfuse v2 → v3, follow the upstream upgrade guide; the image tag bumped from `2` to `3`.
- Upstream docs: https://langfuse.com/self-hosting/docker-compose
- v2 → v3 upgrade: https://langfuse.com/self-hosting/upgrade-guides/upgrade-v2-to-v3