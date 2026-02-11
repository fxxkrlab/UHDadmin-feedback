# UHDadmin Deployment Guide

## Overview

This directory contains Docker Compose configuration for deploying UHDadmin.

### Services

| Service | Description | Port (default) | Profile |
|---------|-------------|----------------|---------|
| postgres | PostgreSQL 15 database | internal | default |
| redis | Redis 7 cache (optional) | internal | redis, full |
| api | FastAPI backend | 8000 | default |
| vben | Vben Admin frontend | 5173 | frontend, full |
| portal | Nuxt Portal + MiniApp | 3000 | frontend, full |

## Quick Start

### 1. Create Environment File

```bash
cp .env.example .env
# Edit .env with your configuration
```

**Required variables:**
- `POSTGRES_PASSWORD` - Database password
- `DATABASE_URL` - Full database connection string
- `JWT_SECRET_KEY` - JWT signing key (min 32 chars)
- `SECRET_KEY` - Application secret key (min 32 chars)

### 2. Start Services

**API + Database only (minimal):**
```bash
docker compose up -d
```

**With Redis:**
```bash
docker compose --profile redis up -d
```

**Full stack (all frontends + Redis):**
```bash
docker compose --profile full up -d
```

**Frontend only (no Redis):**
```bash
docker compose --profile frontend up -d
```

### 3. Verify Deployment

```bash
# Check service status
docker compose ps

# Check API health
curl http://localhost:8000/health

# View logs
docker compose logs -f api
```

## Operations

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f api

# Migration logs (inside container)
docker compose exec api cat /app/logs/migrations.log
```

### Stop Services

```bash
# Stop all
docker compose down

# Stop and remove volumes (WARNING: data loss)
docker compose down -v
```

### Restart Services

```bash
docker compose restart api
```

### Database Access

```bash
# Connect to PostgreSQL
docker compose exec postgres psql -U uhdadmin -d uhdadmin

# Check schema
docker compose exec postgres psql -U uhdadmin -d uhdadmin -c "\dt"

# Check migration status
docker compose exec postgres psql -U uhdadmin -d uhdadmin -c \
    "SELECT filename, applied_at FROM _migrations ORDER BY applied_at DESC LIMIT 10;"
```

### Redis Operations

```bash
# Redis CLI
docker compose exec redis redis-cli

# Check Redis status
docker compose exec redis redis-cli ping
```

## Frontend Services

### Available Frontends

| Frontend | Dockerfile | Port | Technology | Status |
|----------|------------|------|------------|--------|
| vben | `Dockerfile.vben` | 5173 | Vue + Vite + nginx | ✅ Ready |
| portal | `Dockerfile.portal` | 3000 | Nuxt 3 + Node | ✅ Ready |

**Note:** MiniApp is part of Portal, accessible at `/miniapp` route.

### Build Frontends

```bash
# Build all available frontends
docker compose build vben portal

# Build specific frontend
docker compose build vben
```

### Start with Frontends

```bash
# API + Database + Vben + Portal (no Redis)
docker compose --profile frontend up -d

# Full stack with Redis
REDIS_ENABLED=true docker compose --profile redis --profile frontend up -d
```

### Frontend Health Checks

```bash
# Vben (nginx)
curl http://localhost:5173/health
# Returns: healthy

# Portal (Nuxt)
curl -o /dev/null -w "%{http_code}" http://localhost:3000/
# Returns: 200
```

### Frontend Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `VBEN_PORT` | `5173` | Vben Admin exposed port |
| `PORTAL_PORT` | `3000` | Nuxt Portal + MiniApp exposed port |

### Frontend Build Notes

**Vben Admin (`Dockerfile.vben`):**
- Uses pnpm monorepo with turbo
- Builds to static files served by nginx
- Includes gzip compression and 1-year cache headers

**Nuxt Portal (`Dockerfile.portal`):**
- Uses pnpm 9 with frozen lockfile (critical for tailwindcss 3.x)
- Runs as Node.js server (SSR capable)
- npm install was causing tailwindcss v4 issues (fixed)

## Health Checks

### API Health Endpoint

```bash
curl http://localhost:8000/health
```

**Response fields:**
- `status`: "ok" or "degraded"
- `db`: "ok" or "fail"
- `db_latency_ms`: Database response time
- `redis`: "ok", "fail", or "disabled"

### Redis Enabled vs Disabled

**Redis Enabled (`REDIS_ENABLED=true`):**
```json
{
  "status": "ok",
  "version": "v1",
  "db": "ok",
  "db_latency_ms": 5.2,
  "redis": "ok"
}
```

**Redis Disabled (`REDIS_ENABLED=false`):**
```json
{
  "status": "ok",
  "version": "v1",
  "db": "ok",
  "db_latency_ms": 4.8,
  "redis": "disabled"
}
```

## Troubleshooting

### Migration Failed

**Symptoms:** Container exits immediately after starting.

**Solution:**
1. Check migration logs:
   ```bash
   docker compose logs api | grep -A 20 "Running database migrations"
   ```

2. Fix the migration issue in `migrations/` directory.

3. Restart:
   ```bash
   docker compose up -d api
   ```

**Note:** Migrations are idempotent. You can safely restart after fixing issues.

### Database Connection Failed

**Symptoms:** API shows `db: fail` in health check.

**Solution:**
1. Verify PostgreSQL is running:
   ```bash
   docker compose ps postgres
   docker compose logs postgres
   ```

2. Check `DATABASE_URL` in `.env` matches the PostgreSQL credentials.

3. Test connection:
   ```bash
   docker compose exec postgres pg_isready
   ```

### Redis Connection Failed

**Symptoms:** API shows `redis: fail` in health check.

**Solution:**
1. If Redis is required, ensure it's started:
   ```bash
   docker compose --profile redis up -d redis
   ```

2. If Redis is optional, set `REDIS_ENABLED=false` in `.env`.

3. Check Redis logs:
   ```bash
   docker compose logs redis
   ```

### Frontend Not Loading

**Symptoms:** Vben/Portal/MiniApp returns 502 or connection refused.

**Solution:**
1. Ensure API is healthy first:
   ```bash
   curl http://localhost:8000/health
   ```

2. Check frontend container logs:
   ```bash
   docker compose logs vben
   docker compose logs portal
   ```

3. Verify CORS_ORIGINS includes the frontend URL.

## Production Recommendations

1. **Use external PostgreSQL** - For production, use a managed database service.
   Update `DATABASE_URL` to point to external database.

2. **Use external Redis** - For high availability, use Redis Cluster or managed service.
   Update `REDIS_URL` to point to external Redis.

3. **Enable HTTPS** - Place a reverse proxy (nginx, Traefik) in front of services.

4. **Set strong secrets** - Generate random 32+ character strings for:
   - `JWT_SECRET_KEY`
   - `SECRET_KEY`
   - `POSTGRES_PASSWORD`

5. **Resource limits** - Add memory/CPU limits in production:
   ```yaml
   services:
     api:
       deploy:
         resources:
           limits:
             memory: 1G
             cpus: '1.0'
   ```

6. **Log rotation** - Configure Docker log rotation:
   ```yaml
   services:
     api:
       logging:
         driver: "json-file"
         options:
           max-size: "10m"
           max-file: "3"
   ```

## Environment Variables Reference

See `.env.example` for complete list with descriptions.

### Required

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string |
| `JWT_SECRET_KEY` | JWT token signing key |
| `SECRET_KEY` | Application encryption key |
| `POSTGRES_PASSWORD` | Database password |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `REDIS_ENABLED` | `false` | Enable Redis integration |
| `REDIS_URL` | `redis://redis:6379/0` | Redis connection string |
| `ENVIRONMENT` | `PROD` | DEV or PROD mode |
| `LOG_LEVEL` | `INFO` | Logging verbosity |
| `API_PORT` | `8000` | API server port |
| `CORS_ORIGINS` | - | Comma-separated allowed origins |

---

## CI/CD Deployment (DEPLOY-08)

### Overview

UHDadmin uses GitHub Actions for automated CI/CD with the following workflow:

```
Push to main → Build → Test → Push Images → Deploy → Smoke Test → Finalize
                                                ↓ (on failure)
                                            Rollback
```

### Image Tags

All Docker images are tagged with:
- `sha-<GITHUB_SHA>` - Git commit hash (immutable)
- `latest` - Most recent build (mutable)

**Images pushed to GHCR:**
- `ghcr.io/fxxkrlab/uhdadmin-api:<tag>`
- `ghcr.io/fxxkrlab/uhdadmin-vben:<tag>`
- `ghcr.io/fxxkrlab/uhdadmin-portal:<tag>`

### GitHub Actions Workflow

The main workflow (`.github/workflows/deploy.yml`) includes:

| Job | Purpose | Triggers |
|-----|---------|----------|
| `build` | Build Docker images | All pushes to main |
| `test` | Run lint, unit tests, smoke tests | Parallel with build |
| `push` | Push images to GHCR | After build success |
| `deploy` | SSH to server, run deployment | After push success |
| `smoke` | Post-deploy verification | After deploy success |
| `rollback` | Restore previous version | On failure or manual |
| `finalize` | Update last_good_tag | After smoke success |

### Manual Deployment

**Deploy specific tag:**
```bash
# On server
./scripts/deploy_remote.sh sha-abc123
```

**Using workflow dispatch:**
```yaml
# In GitHub Actions UI:
# 1. Go to Actions → Deploy
# 2. Click "Run workflow"
# 3. Enter deploy_tag: sha-abc123
```

### Rollback Procedure

**Automatic rollback** triggers when:
- Smoke test fails after deployment
- Manual rollback requested via workflow dispatch

**Manual rollback:**
```bash
# On server
./scripts/rollback.sh                 # Uses last_good_tag
./scripts/rollback.sh sha-abc123      # Rollback to specific tag
```

**Rollback priority:**
1. Specified tag (if provided)
2. `last_good_tag` (from successful smoke tests)
3. `previous_tag` (from last deployment)

### Post-Deploy Smoke Test

The smoke test (`scripts/post_deploy_smoke.sh`) verifies:

| Check | Expected Result |
|-------|-----------------|
| `/health` | Returns "ok" or "healthy" |
| `/api/v1/public/version` | Returns version info |
| `/api/v1/setup/state` | Accessible |
| `/api/v1/auth/login` | Returns 422/400 (no body) |
| `/api/v1/admin/diagnostics` | Returns 401/403 (auth required) |
| Vben frontend | Returns 200 (if running) |
| Portal frontend | Returns 200 (if running) |

**Run manually:**
```bash
./scripts/post_deploy_smoke.sh
```

### Deployment Data Files

Located in `/opt/uhdadmin/data/`:

| File | Purpose |
|------|---------|
| `current_tag` | Currently deployed tag |
| `previous_tag` | Tag before last deployment |
| `last_good_tag` | Last tag that passed smoke test |
| `deploy_history.jsonl` | Deployment history log |

### Version Endpoint

**Endpoint:** `GET /api/v1/public/version`

**Response:**
```json
{
  "version": "1.0.0",
  "api_version": "v1",
  "git_sha": "abc1234",
  "deploy_tag": "sha-abc1234",
  "environment": "prod",
  "deployed_at": "2025-01-18T10:00:00Z",
  "current_tag": "sha-abc1234",
  "last_good_tag": "sha-abc1234"
}
```

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `DEPLOY_HOST` | Server hostname/IP |
| `DEPLOY_USER` | SSH username |
| `DEPLOY_KEY` | SSH private key |
| `GHCR_TOKEN` | GitHub Container Registry token |

### Server Requirements

- Docker and Docker Compose installed
- SSH access configured
- Deploy directory: `/opt/uhdadmin`
- Data directory: `/opt/uhdadmin/data`
- Scripts: `/opt/uhdadmin/scripts/*.sh` (executable)

---

## Blue/Green & Canary Release (DEPLOY-10)

### Overview

UHDadmin uses APISIX gateway for blue/green and canary deployments:

```
                    ┌─────────────┐
                    │   APISIX    │
                    │   Gateway   │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
     ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
     │   Blue    │   │  Canary   │   │   Green   │
     │  (Stable) │   │ (% split) │   │   (New)   │
     └───────────┘   └───────────┘   └───────────┘
```

### APISIX Gateway Stack

**Start the gateway:**
```bash
cd deploy/apisix
docker compose up -d
```

**Services:**
- `etcd` - Configuration store (port 2379)
- `apisix` - API Gateway (ports 9080, 9443, 9180)

**Initialize routes:**
```bash
./apply.sh
```

### Traffic Control

**Commands:**
```bash
# View current status
./traffic.sh status

# 100% Blue (safe default)
./traffic.sh blue

# 100% Green (after validation)
./traffic.sh green

# Canary mode (e.g., 30% to green)
./traffic.sh canary 30
```

**Emergency rollback:**
```bash
./rollback_blue.sh
```

### Canary Routing Priority

When in canary mode, traffic is routed by priority:

1. **Allowlist** → Always green (specific users for testing)
2. **Denylist** → Always blue (protect specific users)
3. **Consistent Hash** → MD5(user_id) % 100 < canary_percent → green
4. **Default** → blue (no routing key)

**Hash key priority:**
- `X-UHD-UID` header (logged-in user)
- `uhd_did` cookie (device ID for guests)

### Release Console (Admin UI)

Access at: `/admin/ops/release-console`

**Features:**
- View current mode and canary percent
- Quick switch buttons (Blue / Green / Canary)
- Adjust canary percentage with slider
- Manage allowlist/denylist
- Simulate routing decisions
- View audit history

**Safety:**
- All changes require confirm token (5-minute expiry)
- Changes logged to RuntimeLogs with actor ID
- History viewable in console

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/admin/release/traffic` | GET | Get current config |
| `/api/v1/admin/release/traffic` | PUT | Update config (needs token) |
| `/api/v1/admin/release/traffic/request-token` | POST | Request confirm token |
| `/api/v1/admin/release/traffic/history` | GET | View audit history |
| `/api/v1/admin/release/traffic/simulate` | POST | Test routing decision |
| `/api/v1/admin/release/traffic/sync-apisix` | POST | Force sync to APISIX |

### Response Headers

All requests through APISIX include:
- `X-Release-Color: blue|green|mixed`
- `X-Canary-Policy: 100% blue|100% green|canary-30%`
- `X-Request-ID: <uuid>` (for tracing)

### Deployment Workflow

**1. Deploy new version to Green:**
```bash
# On green servers
docker pull ghcr.io/fxxkrlab/uhdadmin-api:sha-abc123
docker compose up -d
```

**2. Canary release:**
```bash
# Start with allowlist
./traffic.sh canary 0
# Add test users to allowlist via API

# Gradually increase
./traffic.sh canary 10
./traffic.sh canary 30
./traffic.sh canary 50
```

**3. Validate:**
- Monitor error rates
- Check X-Release-Color in logs
- Test key user flows

**4. Complete release:**
```bash
./traffic.sh green
```

**5. Rollback if needed:**
```bash
./rollback_blue.sh
```

### Verification Scripts

```bash
# Test blue mode configuration
./verify_blue.sh

# Test green mode configuration
./verify_green.sh

# Test canary mode with distribution
./verify_canary.sh 30
```

### Access Logging

APISIX logs in JSON format to `/logs/access.log`:
```json
{
  "time": "2026-01-18T00:00:00+00:00",
  "client": "203.0.113.10",
  "host": "api.example.com",
  "uri": "/api/v1/health",
  "method": "GET",
  "status": 200,
  "latency": 0.005,
  "upstream": "203.0.113.10:8000",
  "release_color": "blue",
  "trace_id": "abc-123"
}
```

### Troubleshooting

**APISIX not starting:**
```bash
docker compose logs apisix
docker compose logs etcd
```

**Routes not applied:**
```bash
# Check Admin API
curl http://localhost:9180/apisix/admin/routes -H "X-API-KEY : REDACTED"

# Re-apply routes
./apply.sh
```

**Traffic not splitting:**
```bash
# Verify route configuration
./traffic.sh status

# Force sync from database
curl -X POST /api/v1/admin/release/traffic/sync-apisix
```
