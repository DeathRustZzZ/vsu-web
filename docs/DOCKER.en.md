# Docker Configuration Guide

## 1. Overview

This project uses `docker-compose.yml` to run the full stack:
- Infrastructure:
  - `postgres` (`postgres:16`)
  - `redis` (`redis:7`)
- Rust services (built from local Dockerfiles):
  - `api_service`
  - `admin_service`
  - `parser_service`
  - `telegram_service`

Compose file path:
- `docker-compose.yml`

Service Dockerfiles:
- `services/api_service/Dockerfile`
- `services/admin_service/Dockerfile`
- `services/parser_service/Dockerfile`
- `services/telegram_service/Dockerfile`

## 2. Prerequisites

Install:
- Docker Engine
- Docker Compose plugin (`docker compose`)

Check:

```bash
docker --version
docker compose version
```

## 3. Service Map

`postgres`:
- Container: `vsu_postgres`
- Port mapping: `5432:5432`
- Volume: `postgres_data:/var/lib/postgresql/data`
- DB settings:
  - `POSTGRES_USER=admin`
  - `POSTGRES_PASSWORD=admin`
  - `POSTGRES_DB=vsu_db`
- Healthcheck: `pg_isready -U admin -d vsu_db`

`redis`:
- Container: `vsu_redis`
- Port mapping: `6379:6379`
- Healthcheck: `redis-cli ping`

`api_service`:
- Container: `vsu_api_service`
- Build context: project root (`.`)
- Dockerfile: `services/api_service/Dockerfile`
- Port mapping: `8080:8080`
- Environment:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
- Depends on healthy `postgres` and `redis`

`admin_service`:
- Container: `vsu_admin_service`
- Build context: project root (`.`)
- Dockerfile: `services/admin_service/Dockerfile`
- Port mapping: `8081:8081`
- Environment:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
- Depends on healthy `postgres` and `redis`

`parser_service`:
- Container: `vsu_parser_service`
- Build context: project root (`.`)
- Dockerfile: `services/parser_service/Dockerfile`
- Environment:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
- Depends on healthy `postgres` and `redis`

`telegram_service`:
- Container: `vsu_telegram_service`
- Build context: project root (`.`)
- Dockerfile: `services/telegram_service/Dockerfile`
- Environment:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
  - `TELEGRAM_TOKEN` (provide via `.env`)
- Depends on healthy `postgres` and `redis`

## 4. Dockerfile Pattern for Rust Services

All service Dockerfiles use multi-stage builds:

1. `builder` stage (`rust:1.75`)
- Copies whole workspace (`COPY . .`)
- Builds one package with `cargo build --release -p <service_name>`

2. runtime stage (`debian:bookworm-slim`)
- Installs `ca-certificates`
- Copies final binary from builder stage
- Runs the binary as container command

Why workspace-level `COPY` is required:
- Services depend on workspace crates in `crates/*`
- Cargo needs full workspace to resolve local path dependencies

## 5. Environment Variables

`docker-compose.yml` sets most variables directly for containers.

For secrets (like Telegram token), use project `.env` and pass it to compose runtime.

Recommended `.env` example:

```dotenv
TELEGRAM_TOKEN=your_real_token_here
```

Security note:
- Do not commit real tokens/passwords to git history.
- If a token was exposed, rotate it in the source system and update `.env`.

## 6. Start / Stop

Start all services in background:

```bash
docker compose up -d --build
```

Start and stream logs in foreground:

```bash
docker compose up --build
```

Stop containers:

```bash
docker compose stop
```

Stop and remove containers/network:

```bash
docker compose down
```

Stop and remove containers/network + named volumes:

```bash
docker compose down -v
```

## 7. Build Commands

Build all images:

```bash
docker compose build
```

Build one service image:

```bash
docker compose build api_service
docker compose build admin_service
docker compose build parser_service
docker compose build telegram_service
```

Rebuild without cache:

```bash
docker compose build --no-cache
```

## 8. Logs and Diagnostics

All logs:

```bash
docker compose logs -f
```

Single service logs:

```bash
docker compose logs -f api_service
docker compose logs -f admin_service
docker compose logs -f parser_service
docker compose logs -f telegram_service
docker compose logs -f postgres
docker compose logs -f redis
```

Service status:

```bash
docker compose ps
```

Inspect healthchecks:

```bash
docker inspect --format='{{json .State.Health}}' vsu_postgres
docker inspect --format='{{json .State.Health}}' vsu_redis
```

## 9. Access Endpoints and Ports

Host machine access:
- API service: `http://localhost:8080`
- Admin service: `http://localhost:8081`
- Postgres: `localhost:5432`
- Redis: `localhost:6379`

Internal Docker network hostnames (used by services):
- Postgres host: `postgres`
- Redis host: `redis`

## 10. Common Operational Tasks

Restart one service:

```bash
docker compose restart api_service
```

Recreate one service after Dockerfile/env change:

```bash
docker compose up -d --build --no-deps api_service
```

Open shell inside container:

```bash
docker compose exec api_service sh
docker compose exec postgres psql -U admin -d vsu_db
docker compose exec redis redis-cli
```

## 11. Troubleshooting

Port already in use:
- Change host port in `docker-compose.yml` or free local process on that port.

Service exits immediately:
- Check logs: `docker compose logs -f <service>`
- Verify binary name in Dockerfile matches cargo package name.

`telegram_service` fails on startup:
- Ensure `TELEGRAM_TOKEN` is set and valid.

DB/Redis connection errors:
- Confirm `postgres` and `redis` are healthy (`docker compose ps`).
- Ensure service uses internal hosts `postgres` and `redis`, not `localhost`.

Slow builds:
- This is expected on first build because Rust workspace compiles all dependencies.
- Use repeated `docker compose build` to benefit from layer cache.

## 12. File Change Checklist

When adding a new Rust service:

1. Add service package under `services/<new_service>/`.
2. Create `services/<new_service>/Dockerfile` based on existing pattern.
3. Add service section to `docker-compose.yml`:
   - `build.context: .`
   - `build.dockerfile: services/<new_service>/Dockerfile`
   - required `environment`
   - `depends_on` with health checks if DB/Redis needed
4. Rebuild and start:
   - `docker compose up -d --build <new_service>`
