# UHDadmin Deployment Runbook

> Round-DEPLOY-12: Production deployment runbook with domain routing

## Quick Start

### Prerequisites

1. **Docker & Docker Compose** installed
2. **Portainer** (optional, for GUI management)
3. **GHCR access** configured (for pulling images)
4. **Domain names** configured and pointing to server
5. **jq** (JSON processor) - for APISIX route setup:
   ```bash
   # Ubuntu/Debian
   apt-get update && apt-get install -y jq

   # CentOS/RHEL
   yum install -y jq

   # Alpine
   apk add jq
   ```
   Note: `apply.sh` will auto-install jq if not found.

### 1. Clone Repository

```bash
git clone https://github.com/fxxkrlab/UHDadmin.git
cd UHDadmin
```

### 2. Configure Environment

```bash
# Copy environment template
cp deploy/portainer/.env.example .env

# Edit with your values
edgeb .env
```

**Required settings:**
- `POSTGRES_USER`, `POSTGRES_PASSWORD`
- `JWT_SECRET_KEY`, `SECRET_KEY`
- `IMAGE_TAG` (use `latest` or specific `sha-<commit>`)

### 3. Deploy with Portainer

1. Log into Portainer
2. Go to **Stacks** → **Add stack**
3. Upload `deploy/portainer/stack.prod.yml`
4. Configure environment variables
5. Deploy

### 4. Deploy with Docker Compose

```bash
cd deploy/portainer
docker compose -f stack.prod.yml up -d
```

### 5. Verify Deployment

```bash
./scripts/deploy_verify_prod.sh
```

### 6. Complete Setup Wizard (First-time Only)

On first deployment, you must create the initial administrator (sysop) account via the Setup Wizard.

**Important:** The Setup Wizard is **ONLY** available on the Portal domain:

```
https://portal.example.com/setup    ← Setup Wizard (ALLOWED)
https://admin.example.com/setup     ← 403 Forbidden
https://api.example.com/setup       ← 404 Not Found
```

1. Navigate to `https://<your-portal-domain>/setup`
2. Create the first sysop (administrator) account:
   - Username (3-50 characters, alphanumeric)
   - Password (min 8 chars, uppercase + lowercase + digit)
   - Email (optional)
   - Telegram UID (optional)
3. After successful setup, you'll be redirected to login

**Note:** The Setup Wizard can only be used ONCE. After the first sysop is created, the `/setup` endpoint becomes locked and returns a "System Already Initialized" message.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    APISIX Gateway (L7)                      │
│                 Ports: 9080 (HTTP), 9443 (HTTPS)            │
└───────────────────────────┬─────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────▼────┐        ┌─────▼─────┐       ┌─────▼─────┐
   │ API     │        │ Vben      │       │ Portal    │
   │ (Blue)  │        │ (Blue)    │       │ (Blue)    │
   └────┬────┘        └───────────┘       └───────────┘
        │
   ┌────▼────┐
   │ Postgres│
   └─────────┘
```

---

## Volume Storage: Named vs Bind Mounts

UHDadmin supports two volume storage modes. Choose based on your needs:

### Named Volumes (Default, Recommended)

**File:** `deploy/portainer/stack.prod.yml`

```bash
# Deploy with named volumes
docker compose -f deploy/portainer/stack.prod.yml up -d
```

| Pros | Cons |
|------|------|
| Auto-managed by Docker | Stored in /var/lib/docker/volumes/ |
| No setup required | Requires docker commands to access |
| Better performance on some systems | Harder to browse files directly |
| Portable across Docker hosts | Backup requires docker commands |

**Accessing named volume data:**
```bash
# List volumes
bash scripts/volume-tools.sh list

# Inspect a volume
bash scripts/volume-tools.sh inspect postgres

# Export for backup
bash scripts/volume-tools.sh export postgres --output ./backups

# Copy file from volume
docker run --rm -v uhdadmin_apisix_conf:/conf alpine cat /conf/config.yaml
```

### Bind Mounts (Direct Host Access)

**File:** `deploy/portainer/stack.prod.bind.yml`

```bash
# 1. Initialize host directories first
sudo bash scripts/init-host-dirs.sh

# 2. Set base path in .env (optional, default: /opt/uhdadmin)
echo "UHDADMIN_BASE_PATH=/opt/uhdadmin" >> .env

# 3. Deploy with bind mounts
docker compose -f deploy/portainer/stack.prod.bind.yml up -d
```

| Pros | Cons |
|------|------|
| Direct file access (`vim /opt/uhdadmin/...`) | Requires init script before deploy |
| Easy backup with rsync/cp | Permission management needed |
| Clear, predictable paths | Slightly more complex setup |
| Standard tools for management | Path must exist before container start |

**Host directory structure:**
```
/opt/uhdadmin/
├── pg_data/        # PostgreSQL data
├── redis_data/     # Redis persistence
├── etcd_data/      # etcd data
├── apisix/
│   ├── conf/       # APISIX config (config.yaml)
│   └── logs/       # APISIX logs
├── api/
│   └── logs/       # API application logs
└── data/           # Runtime config (boot_config)
```

### Volume Tools

```bash
# List all volumes/directories
bash scripts/volume-tools.sh list

# Inspect specific volume
bash scripts/volume-tools.sh inspect apisix-conf

# Export volume to backup
bash scripts/volume-tools.sh export redis --output ./backups

# Import backup to volume
bash scripts/volume-tools.sh import redis ./backups/redis_backup.tar.gz

# Sync APISIX config from repo
bash scripts/volume-tools.sh apisix-config-sync
```

### Migration: Named → Bind

```bash
# 1. Export all named volumes
bash scripts/volume-tools.sh export postgres
bash scripts/volume-tools.sh export redis
bash scripts/volume-tools.sh export apisix-conf

# 2. Stop current stack
docker compose -f deploy/portainer/stack.prod.yml down

# 3. Initialize bind mount directories
sudo bash scripts/init-host-dirs.sh

# 4. Import data to bind mount paths
bash scripts/volume-tools.sh import postgres ./backups/uhdadmin_postgres_data_*.tar.gz --mode bind

# 5. Deploy with bind mount stack
docker compose -f deploy/portainer/stack.prod.bind.yml up -d
```

---

## Blue/Green Deployment

### Switch Traffic to Green

```bash
cd deploy/apisix
./traffic.sh green 100
```

### Canary Release (10% to Green)

```bash
./traffic.sh green 10
```

### Rollback to Blue

```bash
./rollback_blue.sh
```

---

## Domain Routing Policy (DEPLOY-12)

### Route Priority Design

UHDadmin uses APISIX with host-based routing. Routes are evaluated by priority (higher number = higher priority):

| Priority | Route Type | Example |
|----------|------------|---------|
| 100 | Setup blocking | `/setup` on admin/api/app |
| 50 | Path redirects | `/api/*`, `/uhdadmin/*` |
| 0 | Main service | `/*` catch-all |

### Configurable Domains

| Variable | Description | Example |
|----------|-------------|---------|
| `ROOT_DOMAIN` | Base domain | `example.com` |
| `PORTAL_HOSTS` | Portal domain(s), comma-separated | `portal.example.com,example.com,example.com` |
| `ADMIN_HOST` | Admin (Vben) domain | `admin.example.com` |
| `API_HOST` | API backend domain | `api.example.com` |
| `APP_HOST` | MiniApp domain | `app.example.com` |

### Apply Routes (Parameterized)

Use `apply-routes.sh` for parameterized route configuration:

```bash
cd deploy/apisix

# Basic usage (uses default example.com domain)
./apply-routes.sh

# Custom domains
ROOT_DOMAIN=example.com \
PORTAL_HOSTS=portal.example.com,example.com,www.example.com \
ADMIN_HOST=admin.example.com \
API_HOST=api.example.com \
APP_HOST=app.example.com \
./apply-routes.sh
```

### Setup Wizard Restrictions

The Setup Wizard is **ONLY** accessible on Portal domains. Other domains return errors:

| Domain | `/setup` Response |
|--------|-------------------|
| `portal.*` | **200 OK** (Setup Wizard) |
| `admin.*` | 403 Forbidden |
| `api.*` | 404 Not Found |
| `app.*` | 403 Forbidden |

This is enforced by APISIX routes at priority 100.

### Path-to-Subdomain Redirects (Optional)

Enable optional redirects from path-based URLs to subdomain URLs:

```bash
# Enable redirects (302 temporary by default)
ENABLE_PATH_REDIRECTS=true ./apply-routes.sh

# Use permanent redirects (301)
ENABLE_PATH_REDIRECTS=true REDIRECT_CODE=301 ./apply-routes.sh
```

**Redirect Rules (when enabled):**
| Path | Redirects To |
|------|--------------|
| `/api/*` | `https://api.example.com/*` |
| `/uhdadmin/*` | `https://admin.example.com/*` (prefix stripped) |
| `/uhdadmin/setup` | 403 Forbidden (blocked) |

**302 vs 301 Comparison:**

| Aspect | 302 (Default) | 301 (Permanent) |
|--------|---------------|-----------------|
| Browser caching | No | Indefinite |
| SEO impact | None | Full PageRank transfer |
| Rollback | Easy | Requires cache clear |
| Recommended for | Testing, development | Production finalized |

**Warning:** Use 301 only when domain structure is finalized. Switching back requires users to clear browser cache.

### Route Audit Logging

`apply-routes.sh` writes audit logs to `logs/ac_deploy12_audit_log.json`:

```json
{
  "session": {
    "started_at": "2026-01-20T00:00:00Z",
    "operator": "neo",
    "domains": { ... }
  },
  "entries": [
    {"action": "create_upstream", "target": "api-blue", "result": "success"},
    {"action": "create_route", "target": "api", "result": "success"}
  ]
}
```

### Rollback Routes

To rollback to default configuration:

```bash
# Re-apply with original domains
./apply-routes.sh

# Or manually delete and recreate routes
curl -X DELETE "$APISIX_ADMIN/apisix/admin/routes/path-api-redirect" \
  -H "X-API-KEY : REDACTED"
```

---

## Domain Configuration (11C)

### Via Boot Config API

```bash
curl -X PUT http://localhost:8000/admin/boot-config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "domain_api": "api.example.com",
    "domain_admin": "admin.example.com",
    "domain_portal": "portal.example.com",
    "domain_miniapp": "app.example.com",
    "apisix_admin_url": "http://apisix:9180",
    "apisix_admin_key": "your-admin-key"
  }'
```

### Apply Domains to APISIX

```bash
curl -X POST http://localhost:8000/admin/boot-config/apply-domains \
  -H "Authorization: Bearer $TOKEN"
```

---

## Monitoring

### Health Endpoints

| Service | Endpoint | Expected |
|---------|----------|----------|
| API | `/health` | 200 |
| APISIX | `/apisix/status` | 200 |
| Vben | `/` | 200 |
| Portal | `/` | 200 |

### Logs

```bash
# API logs
docker logs uhdadmin-api -f

# APISIX logs
docker logs uhdadmin-apisix -f

# All containers
docker compose -f stack.prod.yml logs -f
```

---

## Troubleshooting

### Container Won't Start

1. Check logs: `docker logs <container_name>`
2. Verify environment variables
3. Check volume permissions

### Database Connection Failed

1. Verify `DATABASE_URL` format
2. Check PostgreSQL container health
3. Ensure network connectivity

### APISIX Routes Not Working

1. Check etcd is healthy
2. Verify routes with: `curl http://localhost:9180/apisix/admin/routes -H "X-API-KEY : REDACTED"`
3. Re-apply routes: `./deploy/apisix/apply.sh`

### Image Pull Failed

1. Verify GHCR authentication: `docker login ghcr.io`
2. Check image tag exists
3. Try explicit pull: `docker pull ghcr.io/fxxkrlab/uhdadmin-api:latest`

---

## GHCR (GitHub Container Registry) Setup

### Login to GHCR

UHDadmin images are hosted on GitHub Container Registry (GHCR). To pull private images, you need to authenticate.

#### 1. Create a Personal Access Token (PAT)

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token with these scopes:
   - `read:packages` - **Required** for pulling images
   - `write:packages` - Optional, only if you need to push images
3. Copy the token (starts with `GITHUB_TOKEN_REDACTED`)

**Minimum Permission**: `read:packages` is sufficient for pulling production images.

#### 2. Docker Login

```bash
# Method 1: Interactive login
docker login ghcr.io -u YOUR_GITHUB_USERNAME
# Enter your PAT when prompted for password

# Method 2: Non-interactive (CI/scripts)
echo "GITHUB_TOKEN_REDACTED" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Method 3: Environment variable
export GITHUB_TOKEN="GITHUB_TOKEN_REDACTED"
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

#### 3. Verify Login

```bash
# Test pull (should succeed without 401/403)
docker pull ghcr.io/fxxkrlab/uhdadmin-api:latest

# Check credentials stored
cat ~/.docker/config.json | grep ghcr
```

### Image Tag Rules

| Tag Pattern | Example | Description |
|-------------|---------|-------------|
| `latest` | `latest` | Latest stable build on main branch |
| `sha-<commit>` | `sha-5ddd360c9` | Specific commit build |
| `<version>` | `1.0.4` | Release version |
| `stable` | `stable` | Alias for latest release |

### Available Images

| Image | Full URI |
|-------|----------|
| API Backend | `ghcr.io/fxxkrlab/uhdadmin-api` |
| Vben Admin | `ghcr.io/fxxkrlab/uhdadmin-vben` |
| Portal | `ghcr.io/fxxkrlab/uhdadmin-portal` |

### Pull Commands

```bash
# Pull latest (recommended)
docker pull ghcr.io/fxxkrlab/uhdadmin-api:latest
docker pull ghcr.io/fxxkrlab/uhdadmin-vben:latest
docker pull ghcr.io/fxxkrlab/uhdadmin-portal:latest

# Pull specific version
docker pull ghcr.io/fxxkrlab/uhdadmin-api:1.0.4

# Pull specific commit
docker pull ghcr.io/fxxkrlab/uhdadmin-api:sha-5ddd360c9

# Pull all at once
for img in api vben portal; do
  docker pull ghcr.io/fxxkrlab/uhdadmin-$img:latest
done
```

### Portainer Registry Configuration

To use GHCR images in Portainer:

1. Go to **Portainer** → **Registries** → **Add registry**
2. Select **Custom registry**
3. Configure:
   - **Name**: `GHCR`
   - **Registry URL**: `ghcr.io`
   - **Authentication**: Enabled
   - **Username**: Your GitHub username
   - **Password**: Your PAT (with `read:packages` scope)
4. Click **Add registry**

Now you can use GHCR images in Portainer stacks with authentication.

### Troubleshooting GHCR

| Error | Cause | Solution |
|-------|-------|----------|
| `401 Unauthorized` | Not logged in | Run `docker login ghcr.io` |
| `403 Forbidden` | PAT lacks `read:packages` | Regenerate PAT with correct scope |
| `404 Not Found` | Image doesn't exist or no access | Verify image name and org membership |
| `denied: denied` | Private repo, no access | Check you have access to `fxxkrlab/UHDadmin` |

### CI/CD Integration

For automated deployments, store credentials as secrets:

```yaml
# GitHub Actions example
env:
  GHCR_TOKEN: ${{ secrets.GHCR_TOKEN }}

steps:
  - name: Login to GHCR
    run: echo "$GHCR_TOKEN" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
```

---

## Maintenance

### Backup Database

```bash
docker exec uhdadmin-postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB > backup.sql
```

### Restore Database

```bash
cat backup.sql | docker exec -i uhdadmin-postgres psql -U $POSTGRES_USER $POSTGRES_DB
```

### Update Images

```bash
# Pull new images
docker compose -f stack.prod.yml pull

# Recreate containers
docker compose -f stack.prod.yml up -d
```

---

## Security Checklist

- [ ] Change default passwords
- [ ] Generate unique JWT_SECRET_KEY and SECRET_KEY
- [ ] Configure CORS_ORIGINS properly
- [ ] Enable HTTPS via APISIX or reverse proxy
- [ ] Restrict APISIX Admin API (port 9180) access
- [ ] Set up regular database backups
- [ ] Review and rotate API keys periodically

---

## Related Files

| File | Purpose |
|------|---------|
| `deploy/portainer/stack.prod.yml` | Production stack (Named Volumes) |
| `deploy/portainer/stack.prod.bind.yml` | Production stack (Bind Mounts) |
| `deploy/portainer/stack.staging.yml` | Staging stack |
| `deploy/portainer/.env.example` | Environment template |
| `deploy/boot-config.json.example` | Boot config template |
| `deploy/apisix/apply.sh` | APISIX basic route setup |
| `deploy/apisix/apply-routes.sh` | APISIX parameterized routes (DEPLOY-12) |
| `deploy/apisix/traffic.sh` | Traffic control (blue/green/canary) |
| `deploy/apisix/rollback_blue.sh` | Rollback to 100% blue |
| `scripts/init-host-dirs.sh` | Initialize host directories for bind mounts |
| `scripts/volume-tools.sh` | Volume management (list/inspect/export/import) |
| `scripts/deploy_verify_prod.sh` | Deployment verification |
