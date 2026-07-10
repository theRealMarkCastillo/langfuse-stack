# langfuse-stack

Self-hosted [Langfuse](https://langfuse.com) v3 observability stack on a single VM, runnable with one `docker compose up`.

## Quick start

```bash
cp .env.example .env
# edit .env — replace every CHANGEME value, generate ENCRYPTION_KEY with: openssl rand -hex 32

docker compose up -d
open http://localhost:3000
```

Bootstrap an org/project/user on first boot by filling the `LANGFUSE_INIT_*` variables, or skip them and create the admin user from the web UI.

## Services

| Port | Service         | Purpose                                            |
|------|-----------------|----------------------------------------------------|
| 3000 | langfuse-web    | UI + ingestion API (the only public-facing port)   |
| 9090 | minio           | S3-compatible blob store (events, media, exports)  |
| 127.0.0.1:3030 | langfuse-worker | Async processor (health endpoint)            |
| 127.0.0.1:5432 | postgres       | Auth, orgs, projects, API keys                     |
| 127.0.0.1:8123 | clickhouse     | HTTP — traces, observations, scores               |
| 127.0.0.1:9000 | clickhouse     | Native protocol                                    |
| 127.0.0.1:9091 | minio console  | MinIO admin UI                                     |
| 127.0.0.1:6379 | redis          | BullMQ queue, ingestion buffers                    |

Inbound traffic should be restricted to ports `3000` and `9090`; every other service is loopback-only.

## Configuration

All config is read from `.env` (see `.env.example`). Defaults baked into `docker-compose.yml` are insecure placeholders — the stack will start with them, but rotate every `CHANGEME` value before exposing it.

Keys that must agree across variables: `POSTGRES_PASSWORD` and `DATABASE_URL`; `REDIS_AUTH` and the `redis` service command. `ENCRYPTION_KEY` must be exactly 64 hex chars and must never be rotated without re-encrypting stored data.

## More

- `CLAUDE.md` — architecture, request flow, full command list, backup recipes, gotchas
- [Upstream self-hosting docs](https://langfuse.com/self-hosting/docker-compose)
- [v2 → v3 upgrade guide](https://langfuse.com/self-hosting/upgrade-guides/upgrade-v2-to-v3)