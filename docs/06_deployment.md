# 06 — Deployment Guide

## Target Environment

Single VPS running Docker Compose. No Kubernetes, no cloud, no CI/CD for MVP.

## VPS Specifications

Recommended providers (in order of preference):

### Option A: Hetzner Cloud
- Plan: CCX23 (Dedicated CPU)
- Specs: 4 dedicated vCPU, 16 GB RAM, 160 GB SSD
- Price: €32/month (~Rp 560K)
- Region: Falkenstein (FSN1) or Helsinki (HEL1) — stable, fast for Asia
- Pros: Reliable, dedicated CPU, good network
- Cons: Slightly more expensive

### Option B: Contabo
- Plan: VPS L
- Specs: 8 vCPU (shared), 30 GB RAM, 800 GB SSD
- Price: €13/month (~Rp 230K)
- Region: Singapore SIN1 (closest to Indonesia)
- Pros: Cheap, lots of RAM/storage, Singapore region for low latency
- Cons: Shared CPU (occasional throttling)

### Option C: BiznetGio (Indonesian)
- Pros: Local provider, IDR billing, low latency
- Cons: More expensive for similar specs, less mature tooling

**Recommendation:** Start with Contabo VPS L Singapore. Migrate to Hetzner if performance issues.

## OS Setup

Ubuntu 24.04 LTS. Fresh install.

Initial server hardening (run as root after first SSH):

```bash
#!/bin/bash
# scripts/server_init.sh

set -e

# Update system
apt update && apt upgrade -y

# Create deploy user
adduser --gecos "" --disabled-password deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# Disable root SSH
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

# Install firewall
apt install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable

# Install Docker
curl -fsSL https://get.docker.com | sh
usermod -aG docker deploy

# Install fail2ban
apt install -y fail2ban
systemctl enable fail2ban
systemctl start fail2ban

# Install monitoring basics
apt install -y htop tmux git curl wget unzip

# Set timezone (UTC for server, display in Jakarta as needed)
timedatectl set-timezone UTC

echo "Server initialized. Reboot recommended."
```

After this, all subsequent work is done as `deploy` user. Never use root for app deployment.

## Directory Layout on VPS

```
/home/deploy/
├── arena-politika/              # Cloned git repo
│   ├── backend/
│   ├── frontend/
│   ├── workers/
│   ├── infra/
│   │   ├── docker-compose.yml
│   │   ├── docker-compose.prod.yml
│   │   ├── Caddyfile
│   │   └── env/
│   │       └── .env.prod
│   └── ...
├── data/
│   ├── postgres/                # PostgreSQL data volume
│   ├── redis/                   # Redis data volume
│   ├── minio/                   # MinIO data volume
│   └── caddy/                   # Caddy certs and config
└── backups/
    └── postgres/                # Local backup staging
```

The `data/` directory is mounted as Docker volumes. Never delete without backup.

## Docker Compose Architecture

### `infra/docker-compose.yml` (Base)

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - /home/deploy/data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 1gb --maxmemory-policy allkeys-lru
    volumes:
      - /home/deploy/data/redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
    networks:
      - internal

  mongodb:
    image: mongo:7
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
      MONGO_INITDB_DATABASE: arena_news
    volumes:
      - /home/deploy/data/mongodb:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
    networks:
      - internal

  minio:
    image: minio/minio:latest
    restart: unless-stopped
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - /home/deploy/data/minio:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
    networks:
      - internal
      - public

  backend:
    build:
      context: ../backend
      dockerfile: Dockerfile
    restart: unless-stopped
    env_file:
      - env/.env.prod
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      minio:
        condition: service_healthy
    networks:
      - internal
      - public

  worker:
    build:
      context: ../backend
      dockerfile: Dockerfile.worker
    restart: unless-stopped
    env_file:
      - env/.env.prod
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - internal
      - public

  frontend:
    build:
      context: ../frontend
      dockerfile: Dockerfile
    restart: unless-stopped
    networks:
      - public

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /home/deploy/data/caddy:/data
    depends_on:
      - backend
      - frontend
    networks:
      - public

networks:
  internal:
    driver: bridge
  public:
    driver: bridge
```

### `infra/Caddyfile`

```
{
  email admin@arenapolitika.id
}

arenapolitika.id, www.arenapolitika.id {
  encode gzip

  # API routes
  handle /api/* {
    reverse_proxy backend:8000
  }

  # WebSocket (if used)
  handle /ws/* {
    reverse_proxy backend:8000
  }

  # MinIO direct access for clip downloads (signed URLs)
  handle /minio/* {
    uri strip_prefix /minio
    reverse_proxy minio:9000
  }

  # Frontend (catch-all)
  handle {
    reverse_proxy frontend:80
  }

  # Logging
  log {
    output file /data/access.log {
      roll_size 100mb
      roll_keep 7
    }
    format json
  }

  # Headers
  header {
    Strict-Transport-Security "max-age=31536000; includeSubDomains"
    X-Content-Type-Options "nosniff"
    X-Frame-Options "DENY"
    Referrer-Policy "strict-origin-when-cross-origin"
  }

  # Rate limiting (basic)
  rate_limit {
    zone api_rate {
      key {remote_host}
      events 30
      window 1m
    }
  }
}
```

## Environment Variables

`infra/env/.env.example` (committed to repo):

```bash
# === Database ===
POSTGRES_DB=arena_politika
POSTGRES_USER=arena
POSTGRES_PASSWORD=CHANGE_ME_TO_STRONG_PASSWORD
DATABASE_URL=postgresql+asyncpg://arena:CHANGE_ME@postgres:5432/arena_politika

# === Redis ===
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2

# === MongoDB (News Engine) ===
MONGO_USER=arenanews
MONGO_PASSWORD=CHANGE_ME_TO_STRONG_PASSWORD
MONGO_URL=mongodb://arenanews:CHANGE_ME@mongodb:27017/arena_news?authSource=admin

# === News Engine ===
NEWS_DAILY_QUOTA=10
NEWS_LLM_DAILY_BUDGET_USD=0.50
NEWS_RSS_SOURCES=kompas,tempo,detik,cnn_id,antara
NEWS_SCRAPER_USER_AGENT=ArenaPolitika/1.0 (+https://arenapolitika.id/bot)
NEWS_SCRAPER_RATE_LIMIT_PER_DOMAIN=30  # req/min
ADMIN_USER_IDS=1  # Comma-separated user IDs allowed to approve candidates

# === MinIO ===
MINIO_ROOT_USER=arenaminio
MINIO_ROOT_PASSWORD=CHANGE_ME_TO_STRONG_PASSWORD
MINIO_ENDPOINT=minio:9000
MINIO_PUBLIC_ENDPOINT=https://arenapolitika.id/minio
MINIO_BUCKET_AUDIO=audio-uploads
MINIO_BUCKET_CLIPS=clips
MINIO_BUCKET_ASSETS=assets
MINIO_USE_HTTPS=false  # Internal communication

# === JWT ===
JWT_SECRET=CHANGE_ME_TO_RANDOM_64_CHAR_STRING
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# === LLM (OpenRouter) ===
OPENROUTER_API_KEY=sk-or-v1-...
LLM_PRIMARY_MODEL=google/gemini-2.5-flash
LLM_FALLBACK_MODEL=anthropic/claude-haiku-4.5
LLM_TIMEOUT_SECONDS=30
LLM_MAX_RETRIES=2

# === STT (Whisper) ===
OPENAI_API_KEY=sk-...
WHISPER_MODEL=whisper-1
WHISPER_LANGUAGE=id  # Indonesian
WHISPER_TIMEOUT_SECONDS=30

# === Application ===
APP_ENV=production
APP_DEBUG=false
APP_BASE_URL=https://arenapolitika.id
APP_CORS_ORIGINS=https://arenapolitika.id,https://www.arenapolitika.id

# === Rate Limits ===
RATE_LIMIT_DEBATES_PER_DAY=10
RATE_LIMIT_LOGIN_ATTEMPTS=5
RATE_LIMIT_LOGIN_WINDOW_MIN=15

# === Logging ===
LOG_LEVEL=INFO
LOG_FORMAT=json

# === Optional: Monitoring ===
SENTRY_DSN=  # Empty = disabled
```

## Initial Deployment Steps

### 1. Provision VPS
- Order from Contabo or Hetzner
- Point DNS A record `arenapolitika.id` → server IP
- SSH in as root

### 2. Run server init script
```bash
# As root
curl -O https://raw.githubusercontent.com/your-repo/arena-politika/main/scripts/server_init.sh
chmod +x server_init.sh
./server_init.sh
reboot
```

### 3. Switch to deploy user
```bash
ssh deploy@arenapolitika.id
```

### 4. Clone repo
```bash
git clone https://github.com/your-org/arena-politika.git
cd arena-politika
```

### 5. Configure environment
```bash
cp infra/env/.env.example infra/env/.env.prod
nano infra/env/.env.prod  # Fill in real values, especially passwords and API keys
```

Generate strong values:
```bash
# JWT_SECRET
openssl rand -hex 32

# DB passwords
openssl rand -base64 24
```

### 6. Initialize MinIO buckets
```bash
docker compose -f infra/docker-compose.yml up -d minio
sleep 10
docker compose -f infra/docker-compose.yml exec minio mc alias set local http://localhost:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
docker compose -f infra/docker-compose.yml exec minio mc mb local/audio-uploads
docker compose -f infra/docker-compose.yml exec minio mc mb local/clips
docker compose -f infra/docker-compose.yml exec minio mc mb local/assets

# Set lifecycle: auto-delete audio after 24h
docker compose -f infra/docker-compose.yml exec minio mc ilm add --expiry-days 1 local/audio-uploads
```

### 7. Build and start everything
```bash
cd infra
docker compose up -d --build
```

### 8. Run database migrations
```bash
docker compose exec backend alembic upgrade head
```

### 9. Seed initial data
```bash
docker compose exec backend python -m scripts.seed
```

### 10. Verify
```bash
docker compose ps
curl https://arenapolitika.id/api/health
```

Expected: All services `Up (healthy)`. Health endpoint returns `200 OK`.

## Updating the Application

For routine updates after initial deploy:

```bash
ssh deploy@arenapolitika.id
cd arena-politika
git pull origin main
cd infra
docker compose build backend worker frontend
docker compose up -d
docker compose exec backend alembic upgrade head  # If migrations changed
```

For zero-downtime, a more sophisticated process is needed (blue-green deploy). Defer to v2.

## Backup

### Daily PostgreSQL backup

`scripts/backup_postgres.sh` (run via cron):

```bash
#!/bin/bash
set -e

BACKUP_DIR=/home/deploy/backups/postgres
TIMESTAMP=$(date -u +"%Y%m%d_%H%M%S")
FILENAME="arena_${TIMESTAMP}.sql.gz"

mkdir -p $BACKUP_DIR

cd /home/deploy/arena-politika/infra
docker compose exec -T postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB | gzip > $BACKUP_DIR/$FILENAME

# Upload to off-VPS storage via Restic
restic backup $BACKUP_DIR/$FILENAME

# Keep last 7 local
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "Backup complete: $FILENAME"
```

Cron entry:
```
0 2 * * * /home/deploy/arena-politika/scripts/backup_postgres.sh >> /home/deploy/backups/backup.log 2>&1
```

### Configuring Restic for off-VPS backup

```bash
# Backblaze B2 example
export B2_ACCOUNT_ID=...
export B2_ACCOUNT_KEY=...
export RESTIC_REPOSITORY=b2:arena-politika-backups:/postgres
export RESTIC_PASSWORD=...

restic init  # First time only
```

## Monitoring

For MVP, basic monitoring only:

### Logs

All container logs go to host:
```bash
docker compose logs -f --tail=100 backend
docker compose logs -f --tail=100 worker
docker compose logs -f --tail=100 caddy
```

Caddy access logs in `/home/deploy/data/caddy/access.log`.

### Health check script

`scripts/health_check.sh` (run every 5 minutes):

```bash
#!/bin/bash
URL=https://arenapolitika.id/api/health
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" $URL)

if [ "$RESPONSE" != "200" ]; then
  echo "ALERT: Health check failed with code $RESPONSE at $(date)" | mail -s "Arena Politika Down" admin@arenapolitika.id
fi
```

Cron:
```
*/5 * * * * /home/deploy/arena-politika/scripts/health_check.sh
```

### Disk space alert

```bash
# scripts/disk_check.sh
USAGE=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $USAGE -gt 80 ]; then
  echo "Disk usage is $USAGE%" | mail -s "Disk Alert" admin@arenapolitika.id
fi
```

Cron:
```
0 */6 * * * /home/deploy/arena-politika/scripts/disk_check.sh
```

## Rollback Procedure

If a deploy breaks production:

```bash
cd /home/deploy/arena-politika
git log --oneline -5  # Find last good commit
git checkout <last-good-sha>
cd infra
docker compose build
docker compose up -d
# If migration was the issue:
docker compose exec backend alembic downgrade -1
```

Document the issue in `incidents/` folder for postmortem.

## Cost Tracking

Set up monthly review checklist:
- [ ] VPS bill
- [ ] Domain renewal
- [ ] OpenRouter usage report
- [ ] OpenAI Whisper usage report
- [ ] Backblaze B2 storage cost
- [ ] Total: should be < Rp 6 juta/month

If trending higher, profile which calls are expensive and optimize.
