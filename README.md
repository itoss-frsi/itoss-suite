# ITOSS Suite — Quick Start

Spin up the full ITOSS stack (DB, Manager, Collector, Reporting, DS, Frontend, Job Scheduler) with Docker Compose.

##

```bash
cp .env.example .env
docker compose pull
docker compose up -d
docker compose ps    # wait for “healthy”
open http://localhost   # (or your FRONTEND_PORT)
```

## Requirements

- Linux, macOS, or Windows 10/11
- Docker 24+ and `docker compose` (v2)
- ~2 vCPU, 4–8 GB RAM (dev)
- Ports free on host: 80, 5432, 8079, 8080, 8081, 8084, 8085 (configurable)
- ~5 GB disk for images + DB volume

## Services

| Service            | Image                                | Healthcheck / Ready      | Host→Container |
|--------------------|--------------------------------------|--------------------------|----------------|
| itoss-db           | itosssoftware/itoss-db:v8            | `pg_isready` + init flag | 5432→5432      |
| itoss-manager      | itosssoftware/itoss-manager:v8       | `GET /stats`             | ${MANAGER_PORT:-8080}→8080 |
| itoss-collector    | itosssoftware/itoss-collector:v8     | depends on DB/Manager    | ${COLLECTOR_PORT:-8081}→8081 |
| itoss-reporting    | itosssoftware/itoss-reporting:v8     | depends on DB            | ${REPORTING_PORT:-8079}→8079 |
| itoss-ds           | itosssoftware/itoss-ds:v8            | depends on DB            | ${DS_PORT:-8085}→8085 |
| itoss-jobscheduler | itosssoftware/itoss-jobscheduler:v8  | depends on DB            | ${JOBSCHEDULER_PORT:-8084}→8084 |
| itoss-frontend     | itosssoftware/itoss-frontend:v8      | depends on Manager/Rep.  | ${FRONTEND_PORT:-80}→80 |

Data persists in `./data` (mapped into the Postgres container).

## Usage

1. **Configure environment** (create `.env`)
   ```bash
   cp .env.example .env
   # edit if needed (ports/credentials)
   ```

2. **Start the stack**
   ```bash
   docker compose pull
   docker compose up -d
   docker compose ps
   ```

3. **Verify health**
   ```bash
   curl -f http://localhost:${MANAGER_PORT:-8080}/stats
   ```

4. **Access**
   - Frontend: <http://localhost:${FRONTEND_PORT:-80}>
   - Manager API: <http://localhost:${MANAGER_PORT:-8080}>
   - Reporting: <http://localhost:${REPORTING_PORT:-8079}>
   - Collector: <http://localhost:${COLLECTOR_PORT:-8081}>
   - Job Scheduler: <http://localhost:${JOBSCHEDULER_PORT:-8084}>
   - DS: <http://localhost:${DS_PORT:-8085}>

## Common tasks

```bash
# logs
docker compose logs -f itoss-manager

# update
docker compose pull && docker compose up -d

# stop / start
docker compose stop
docker compose start

# wipe & re-seed (dev only)
docker compose down -v
rm -rf ./data
docker compose up -d
```

## Backups (dev)

```bash
docker exec -t itoss-db pg_dump -U "${ITOSS_USER:-itoss}" -d "${ITOSS_DB:-itossdb}" > itossdb_$(date +%Y%m%d).sql
# restore
cat itossdb_20250101.sql | docker exec -i itoss-db psql -U "${ITOSS_USER:-itoss}" -d "${ITOSS_DB:-itossdb}"
```

## Troubleshooting

- **Port in use:** change the **left side** of a port mapping in `.env` (e.g., `18080:8080`).  
- **Manager not healthy:** `docker compose logs itoss-manager` and verify DB URL/creds.  
- **DB init stuck:** check `itoss-db` logs; ensure `ITOSS_DB_DUMPFILE` exists in the image and `.itoss_init_done` marker appears.  
- **Permissions on `./data`:** `sudo chown -R $USER:$USER ./data` (Linux/macOS).

## Production notes

- Use *Docker secrets* or an external manager for credentials.
- Put a TLS reverse proxy in front (Nginx/Traefik/Caddy).
- Add backups/monitoring; set resource limits as needed.
- Point `*_DB_URL` to your managed Postgres if externalizing DB.
