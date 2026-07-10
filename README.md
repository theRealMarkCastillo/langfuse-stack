# langfuse-stack

Self-hosted [Langfuse](https://langfuse.com) v3 observability stack on a single VM, runnable with one `docker compose up`.

## Quick start

```bash
cp .env.example .env
# (optional) edit .env to customize passwords and secrets

docker compose up -d
open http://localhost:3000
```

Bootstrap an org/project/user on first boot by filling the `LANGFUSE_INIT_*` variables, or skip them and create the admin user from the web UI.

## Services

| Port  | Service         | Purpose                                            |
|-------|-----------------|----------------------------------------------------|
| 3000  | langfuse-web    | UI + ingestion API (the only public-facing port)   |
| 9090  | minio           | S3-compatible blob store (events, media, exports)  |

All other services (`langfuse-worker`, `postgres`, `clickhouse`, `redis`, the minio console) are reachable only over the Compose network — no host ports are published for them. Use `docker compose exec <service>` for CLI access — e.g. `docker compose exec postgres psql -U postgres`. The MinIO console (container port `9001`) is not published; add a port mapping in `docker-compose.yml` if you need it for debugging.

## Configuration

All config is read from `.env` (see `.env.example`). Defaults baked into `docker-compose.yml` are safe for local dev but insecure for production — rotate secrets with `openssl rand -hex 32` before exposing the stack to the internet.

Keys that must agree across variables: `POSTGRES_PASSWORD` and `DATABASE_URL`; `REDIS_AUTH` and the `redis` service command. `ENCRYPTION_KEY` must be exactly 64 hex chars and must never be rotated without re-encrypting stored data.

## More

- `AGENTS.md` (also available as `CLAUDE.md`) — architecture, request flow, full command list, backup recipes, gotchas
- [Upstream self-hosting docs](https://langfuse.com/self-hosting/docker-compose)
- [v2 → v3 upgrade guide](https://langfuse.com/self-hosting/upgrade-guides/upgrade-v2-to-v3)