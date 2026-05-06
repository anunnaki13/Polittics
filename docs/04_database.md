# 04 — Database Schema

## Overview

PostgreSQL 16. SQLAlchemy 2.0 ORM. Alembic for migrations.

All tables use UUID primary keys (except `users` which uses serial integer for simpler JWT handling).

All timestamps are `TIMESTAMPTZ` stored in UTC.

Soft-delete is **not** used. Deletes are hard. If we need to preserve data for audit, we copy to an audit table.

---

## Tables

### `users`

User accounts.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Used in JWT |
| `email` | `VARCHAR(255)` | UNIQUE, NOT NULL | Lowercased before insert |
| `username` | `VARCHAR(32)` | UNIQUE, NOT NULL | 3-32 chars, alphanumeric + underscore |
| `password_hash` | `VARCHAR(255)` | NOT NULL | Argon2 hash |
| `display_name` | `VARCHAR(64)` | NULL | Optional, defaults to username |
| `xp` | `INTEGER` | NOT NULL DEFAULT 0 | Total XP, never decreases |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |
| `last_login_at` | `TIMESTAMPTZ` | NULL | Updated on each successful login |
| `is_active` | `BOOLEAN` | NOT NULL DEFAULT TRUE | False = banned/disabled |

Indexes:
- `idx_users_email` on `email`
- `idx_users_username` on `username`

---

### `topics`

Debate topics from two sources: news engine (auto-generated, see `specs/08_news_engine.md`) and evergreen seed (manual fallback).

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `UUID` | PK | Generated via `gen_random_uuid()` |
| `motion` | `TEXT` | NOT NULL | The debate statement, e.g., "Subsidi BBM harus dicabut total" |
| `category` | `VARCHAR(32)` | NOT NULL | One of: `ekonomi`, `sosial`, `hukum`, `lingkungan`, `pendidikan`, `kesehatan`, `politik` |
| `difficulty` | `VARCHAR(16)` | NOT NULL | One of: `mudah`, `sedang`, `sulit` |
| `context` | `TEXT` | NULL | Optional background info shown to user |
| `source` | `VARCHAR(32)` | NOT NULL | `news_engine` or `evergreen_seed` |
| `source_ref` | `TEXT` | NULL | MongoDB ObjectId of `topic_candidates` doc (for news_engine source) |
| `is_active` | `BOOLEAN` | NOT NULL DEFAULT TRUE | False = retired topic |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |

Indexes:
- `idx_topics_active_category` on `(is_active, category)` for filtering
- `idx_topics_source` on `source` for analytics

---

### `personas`

The 3 AI debate personas. Static data, seeded once.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `UUID` | PK | |
| `slug` | `VARCHAR(32)` | UNIQUE, NOT NULL | E.g., `si_profesor`, `si_pak_rt`, `si_aktivis` |
| `name` | `VARCHAR(64)` | NOT NULL | Display name |
| `archetype` | `VARCHAR(64)` | NOT NULL | E.g., "Intelektual", "Populis", "Idealis" |
| `description` | `TEXT` | NOT NULL | Short description shown to user |
| `tagline` | `VARCHAR(128)` | NOT NULL | Catchphrase, e.g., "Data adalah senjata terbaik" |
| `difficulty` | `VARCHAR(16)` | NOT NULL | `mudah`, `sedang`, `sulit` |
| `system_prompt` | `TEXT` | NOT NULL | LLM system prompt (long) |
| `target_score` | `INTEGER` | NOT NULL | Score user must beat to win (60, 70, 80) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |

Note: 3 rows total for MVP. See `prompts/opponent_prompts.md` for content.

---

### `debates`

A single debate session.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `UUID` | PK | |
| `user_id` | `BIGINT` | FK → users.id, NOT NULL | |
| `topic_id` | `UUID` | FK → topics.id, NOT NULL | |
| `persona_id` | `UUID` | FK → personas.id, NOT NULL | |
| `position` | `VARCHAR(16)` | NOT NULL | `PRO` or `KONTRA` |
| `status` | `VARCHAR(32)` | NOT NULL DEFAULT 'pending' | See status flow below |
| `audio_url` | `TEXT` | NULL | MinIO URL, deleted after processing |
| `transcript` | `TEXT` | NULL | Filled by Whisper |
| `transcript_confidence` | `REAL` | NULL | Whisper avg confidence 0-1 |
| `audio_duration_sec` | `REAL` | NULL | Actual recording length |
| `opponent_response` | `TEXT` | NULL | AI counter-argument text |
| `clip_url` | `TEXT` | NULL | MinIO URL of generated MP4 |
| `error_message` | `TEXT` | NULL | If status='failed' |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |
| `submitted_at` | `TIMESTAMPTZ` | NULL | When user uploaded audio |
| `completed_at` | `TIMESTAMPTZ` | NULL | When all processing finished |

Indexes:
- `idx_debates_user_created` on `(user_id, created_at DESC)` for history
- `idx_debates_status` on `status` for finding pending tasks

Status flow:
```
pending → transcribing → scoring → generating_response → generating_clip → complete
                                                                         ↓
                                                                      failed
```

---

### `scores`

Score breakdown for a completed debate. One row per debate.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `UUID` | PK | |
| `debate_id` | `UUID` | FK → debates.id, UNIQUE, NOT NULL | One score per debate |
| `logika` | `REAL` | NOT NULL | 0.0 - 10.0 |
| `data` | `REAL` | NOT NULL | 0.0 - 10.0 |
| `emosi` | `REAL` | NOT NULL | 0.0 - 10.0 |
| `konsistensi` | `REAL` | NOT NULL | 0.0 - 10.0 |
| `retorika` | `REAL` | NOT NULL | 0.0 - 10.0 |
| `total` | `REAL` | NOT NULL | 0.0 - 100.0 (weighted sum × 10) |
| `feedback_logika` | `TEXT` | NOT NULL | 1-2 sentences |
| `feedback_data` | `TEXT` | NOT NULL | |
| `feedback_emosi` | `TEXT` | NOT NULL | |
| `feedback_konsistensi` | `TEXT` | NOT NULL | |
| `feedback_retorika` | `TEXT` | NOT NULL | |
| `feedback_overall` | `TEXT` | NOT NULL | 2-4 sentences summary |
| `result` | `VARCHAR(16)` | NOT NULL | `MENANG`, `KALAH`, `SERI` |
| `xp_earned` | `INTEGER` | NOT NULL | XP awarded for this debate |
| `model_used` | `VARCHAR(64)` | NOT NULL | E.g., "google/gemini-2.5-flash" |
| `tokens_input` | `INTEGER` | NOT NULL | For cost tracking |
| `tokens_output` | `INTEGER` | NOT NULL | For cost tracking |
| `latency_ms` | `INTEGER` | NOT NULL | Scoring API call duration |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |

Indexes:
- `idx_scores_debate` on `debate_id`

---

### `rate_limits`

Server-side rate limit tracking. Most rate limits use Redis (faster), but we keep daily session counts here for analytics + ban tracking.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | |
| `user_id` | `BIGINT` | FK → users.id, NOT NULL | |
| `date` | `DATE` | NOT NULL | UTC date |
| `sessions_count` | `INTEGER` | NOT NULL DEFAULT 0 | |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |

Constraints:
- UNIQUE on `(user_id, date)`

---

### `audit_log` (optional, can be deferred)

For tracking suspicious or important events.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | |
| `user_id` | `BIGINT` | FK → users.id, NULL | NULL for system events |
| `event_type` | `VARCHAR(64)` | NOT NULL | E.g., `login_failed`, `rate_limit_hit`, `content_flagged` |
| `metadata` | `JSONB` | NOT NULL DEFAULT '{}' | Free-form context |
| `ip_hash` | `VARCHAR(64)` | NULL | SHA-256 of IP, never plaintext |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | |

Indexes:
- `idx_audit_user_event` on `(user_id, event_type, created_at DESC)`

---

## Relationships Summary

```
users (1) ─── (M) debates
topics (1) ── (M) debates
personas (1) ─ (M) debates
debates (1) ─ (1) scores

users (1) ─── (M) rate_limits
users (1) ─── (M) audit_log
```

---

## Migration Strategy

Alembic auto-generates migrations from SQLAlchemy models. Process:

1. Modify SQLAlchemy model in `app/<module>/models.py`
2. Run `alembic revision --autogenerate -m "add new column foo to debates"`
3. Review the generated migration in `migrations/versions/<id>_add_new_column_foo.py`
4. Edit if needed (auto-gen sometimes misses things)
5. Run locally: `alembic upgrade head`
6. Test on staging
7. Commit migration file
8. Deploy: migration runs automatically via entrypoint script

Never edit committed migrations. If you need to change something, write a new migration.

---

## Seed Data

Required seeds (loaded on first deploy):

### `personas` table
3 rows: Si Profesor, Si Pak RT, Si Aktivis 98. See `prompts/opponent_prompts.md` for content.

### `topics` table — Evergreen Seed
10-15 rows manually written by owner in `backend/scripts/evergreen_topics.py`. Covers all 7 categories, mostly `mudah`/`sedang`. Provides day-one fallback before news engine produces output. All evergreen rows have `source='evergreen_seed'`.

Auto-generated topics (`source='news_engine'`) are inserted later via the news engine's admin approval endpoints — see `specs/08_news_engine.md`.

Seed script: `backend/scripts/seed.py`. Idempotent (safe to run multiple times).

---

## MongoDB Collections (News Engine)

Separate from PostgreSQL, used by news engine pipeline (full schema in `specs/08_news_engine.md`):

- `news_articles` — Raw fetched articles with status flag
- `topic_candidates` — LLM-generated motion candidates pending review

Backed up daily alongside PostgreSQL via mongodump.

---

## Backup Strategy

- Daily `pg_dump` at 02:00 UTC, compressed with gzip
- Backup uploaded to off-VPS storage (Backblaze B2 or similar) via Restic
- Retention: 7 daily, 4 weekly, 3 monthly
- Test restore monthly (manual, document in checklist)

---

## Things NOT in Schema (Yet)

For clarity, deferred to v2:

- `parties` table — clan/party system
- `party_members` table — many-to-many user-party
- `elections` table — virtual elections
- `votes` table — election votes
- `parliament_sessions` table — RUU voting
- `cognitive_compass` data — ideology profile (could be a JSONB column on users when added)

When v2 is approved, these will be designed separately.
