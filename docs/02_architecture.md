# 02 — System Architecture

## High-Level Architecture

The MVP runs as a set of Docker containers on a single VPS, fronted by Caddy for HTTPS and reverse proxying.

```
                     Internet
                        │
                     [Caddy]  (HTTPS, reverse proxy, auto-cert)
                    /        \
         [Frontend]            [Backend API]
         (React/Vite)          (FastAPI)
         static files               │
                                    ├──> [PostgreSQL] (users, debates, scores, topics)
                                    ├──> [MongoDB]    (news articles, topic candidates)
                                    ├──> [Redis]      (sessions, rate limit, queue)
                                    ├──> [MinIO]      (audio files, video clips)
                                    │
                                    └──> publishes jobs to Celery queue
                                                     │
                                              [Celery Workers + Beat]
                                              ├──> Whisper API (STT)
                                              ├──> OpenRouter   (LLM scoring + news pipeline)
                                              ├──> FFmpeg       (clip generation)
                                              ├──> RSS/HTTP     (news polling)
                                              └──> writes back to DBs + MinIO
```

## Components

### Caddy (Reverse Proxy)

Caddy is chosen over Nginx because it auto-provisions HTTPS certificates via Let's Encrypt with zero config. For MVP, this saves a day of nginx + certbot setup.

It routes:
- `/` and `/static/*` → Frontend container (React build served by Caddy itself)
- `/api/*` → Backend container
- `/uploads/*` → Direct stream from MinIO via signed URLs (NOT proxied through Caddy for large files)
- `/ws/*` → Backend WebSocket endpoint

### Frontend (React + Vite)

Single-page application built with Vite, served as static files. No SSR for MVP.

Key flows:
- Auth (register, login, logout)
- Debate session (record audio, upload, poll for result)
- Result display (scores, feedback, clip)
- History page (list past debates)

State management: Zustand for client state, TanStack Query for server state.

Audio handling: MediaRecorder API for recording, Wavesurfer.js for visualization. Audio recorded as WebM/Opus (smaller than WAV).

Production build is around 200-400KB gzipped. Served by Caddy with proper cache headers.

### Backend API (FastAPI)

Single FastAPI service handling all HTTP/WebSocket traffic.

Modules:
- `auth/` — User registration, login, JWT issuance
- `debates/` — Create debate session, upload audio, get result
- `topics/` — List topics (sourced from news engine + evergreen seed)
- `personas/` — List AI personas
- `users/` — Profile, history, basic stats
- `clips/` — Get clip URL, share metadata

Async-first using FastAPI + asyncpg. SQLAlchemy 2.0 ORM.

Authentication: JWT with refresh tokens. Tokens in HTTPOnly cookies (more secure than localStorage).

### PostgreSQL

Primary database for relational data. See `04_database.md` for schema.

Critical tables:
- `users` — Account info
- `debates` — Each debate session
- `scores` — Score breakdown per debate
- `topics` — Curated topic list
- `personas` — 3 AI personas

Backup: nightly `pg_dump` cron via Restic to off-VPS storage.

### Redis

Used for:
- Session storage (refresh token blacklist)
- Rate limiting (per-user and per-IP counters)
- Celery task broker
- Cache (topic list, persona list, leaderboard if added later)

Single instance, no clustering for MVP.

### MinIO

S3-compatible object storage running on the same VPS.

Buckets:
- `audio-uploads/` — Raw audio recordings (auto-deleted after 24h via lifecycle policy)
- `clips/` — Generated highlight videos (kept indefinitely)
- `assets/` — Static assets used in clip generation (logos, watermarks, fonts)

Frontend uploads audio via signed URLs (presigned PUT). Backend never receives audio directly — saves bandwidth.

### MongoDB

Non-relational store for the news engine pipeline (see `specs/08_news_engine.md`). Kept separate from PostgreSQL because:
- Article schemas vary across sources
- High write volume from RSS polling (~hundreds/day)
- No complex joins needed — document model fits

Collections:
- `news_articles` — Raw fetched articles with status flag
- `topic_candidates` — LLM-generated motion candidates pending review

Single instance, no replica set for MVP. Daily backup same as PostgreSQL.

### Celery Workers

Background workers for slow operations. Run with Celery Beat for scheduled tasks.

Debate pipeline tasks:
- `transcribe_audio(debate_id)` — Calls Whisper API, saves transcript
- `score_debate(debate_id)` — Calls LLM, computes 5-dimension score, saves
- `generate_opponent_response(debate_id)` — LLM generates AI counter-argument
- `generate_clip(debate_id)` — FFmpeg creates 15-25s highlight video
- `cleanup_audio(debate_id)` — Deletes raw audio after processing

News engine tasks (see `specs/08_news_engine.md`):
- `poll_rss()` — Beat-scheduled hourly, fetches RSS feeds into MongoDB
- `scrape_article(article_id)` — Fetches full article text when RSS summary insufficient
- `filter_articles()` — Beat-scheduled every 6h, filters relevant articles
- `sanitize_and_generate_motion(article_id)` — LLM pipeline (sanitize → motion)
- `target_quota()` — Beat-scheduled daily, ensures 5-10 fresh candidates exist

Single worker process for MVP (3-4 concurrent tasks). Scale to multiple workers when load demands.

## Data Flow: One Debate Session

This is the canonical flow that 80% of code in the MVP supports.

### Step 1: User Selects Topic and Persona

Frontend calls:
```
GET  /api/topics
GET  /api/personas
POST /api/debates  { topic_id, persona_id, position: "PRO" | "KONTRA" }
```

Backend creates a `debates` row with status=`pending` and returns `debate_id` plus a presigned MinIO upload URL.

### Step 2: User Records Audio

Frontend uses MediaRecorder API. User clicks record, speaks, clicks stop (or auto-stops at 60s).

Audio blob is uploaded directly to MinIO using the presigned URL. Frontend then notifies backend:
```
POST /api/debates/{id}/submit
```

Backend updates `debates.status = 'transcribing'` and enqueues Celery tasks.

### Step 3: Background Processing (Async)

Celery worker:
1. `transcribe_audio` — Downloads audio from MinIO, calls Whisper API, saves transcript to `debates.transcript`. Updates status to `scoring`.
2. `score_debate` — Sends transcript + topic + position + persona to LLM with scoring prompt. Parses JSON response. Inserts into `scores` table. Updates status to `generating_response`.
3. `generate_opponent_response` — Sends user transcript + persona prompt to LLM. Saves AI response text to `debates.opponent_response`. Updates status to `generating_clip`.
4. `generate_clip` — FFmpeg combines audio + visual template + text overlays. Uploads MP4 to MinIO. Saves URL to `debates.clip_url`. Updates status to `complete`.
5. `cleanup_audio` — After all above succeed, deletes raw audio from MinIO.

If any step fails, status becomes `failed` with error logged. User sees retry button.

### Step 4: Frontend Polls for Result

While processing, frontend polls every 2 seconds:
```
GET /api/debates/{id}/status
```

When status is `complete`, frontend fetches:
```
GET /api/debates/{id}
```

This returns the full debate object with scores, feedback, opponent response, and clip URL.

### Step 5: User Sees Result

Frontend renders the result page. User can:
- View detailed scores
- Read feedback
- Read AI opponent response
- Play/download the generated clip
- Click share buttons (which open native share dialog with the clip)

## Resource Sizing for MVP VPS

Target: 50 concurrent users, 200 sessions/day.

Single VPS specs:
- 4-8 vCPU
- 16 GB RAM
- 200 GB SSD
- 1 Gbps network

Recommended providers:
- Hetzner CCX23 (€32/month, 4 vCPU dedicated, 16GB RAM)
- Contabo VPS L (€13/month, 8 vCPU, 30GB RAM, but burst CPU)

Memory allocation:
- PostgreSQL: 3 GB
- MongoDB: 1.5 GB
- Redis: 1 GB
- MinIO: 1 GB
- Backend (FastAPI, 4 workers): 2 GB
- Celery workers (3 processes): 2 GB
- Frontend (static, served by Caddy): negligible
- Caddy: 200 MB
- OS overhead: 2 GB
- Headroom: 3 GB

## Failure Modes and Handling

### LLM API Down

OpenRouter has multiple model providers. If primary (Gemini Flash) fails, fall back to Claude Haiku, then DeepSeek free tier.

If ALL providers down (rare): mark debate as `failed`, allow user to retry. Email admin alert.

### Whisper API Down

OpenAI Whisper has good uptime but does fail. Fall back to local Whisper-tiny model running in Celery worker (slower, less accurate, but functional).

If both down: queue for retry with exponential backoff. Admin alert if backlog > 50 sessions.

### MinIO Disk Full

Set up Caddy + Prometheus alert at 80% disk. Lifecycle policy auto-deletes audio after 24h. Clips beyond 90 days can be archived to cheaper external storage if needed.

### Database Down

If PostgreSQL fails, the entire app is down. Acceptable for MVP. Set up systemd auto-restart and monitoring. Daily backups so worst case is 24h data loss.

### High Load Spike

If sudden viral traffic:
1. Caddy rate-limits to 30 requests/min per IP
2. Backend rejects new debate creation when Celery queue > 100 pending tasks
3. Frontend shows "Sistem sedang ramai, coba lagi 1 menit" message

These limits are intentional. Better to fail gracefully than degrade for everyone.

## Security Layers

See `08_security.md` for detail. Summary:

- HTTPS everywhere via Caddy
- Argon2 password hashing
- JWT with short expiry (15 min) + refresh token (7 days)
- Rate limiting at Caddy (IP) and FastAPI (user) level
- Input validation on all endpoints (Pydantic)
- SQL injection: prevented by SQLAlchemy ORM
- XSS: prevented by React's default escaping
- CSRF: not relevant since we use Bearer tokens, not cookies for auth
- Audio file size limit: 5MB hard cap
- Voice content scanning: LLM-based moderation on transcript before scoring

## What's NOT in This Architecture

For clarity:

- No CDN. Caddy serves static files. Adequate for MVP.
- No load balancer. Single backend instance.
- No service mesh. Direct container-to-container communication.
- No message queue beyond Celery/Redis. Adequate for MVP.
- No separate auth service. Built into FastAPI app.
- No observability stack (Prometheus, Grafana). Just logs to file + occasional `htop` check.
- No CI/CD pipeline initially. Manual deploy via SSH + git pull. Add CI in week 8.
