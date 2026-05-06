# 05 — API Design

## Conventions

- Base URL: `/api`
- All endpoints return JSON
- Authentication: JWT Bearer token in `Authorization` header (or HTTPOnly cookie for browser)
- Errors follow standard format (see below)
- All endpoints documented via FastAPI's auto-generated OpenAPI at `/api/docs`

### Standard Error Response

```json
{
  "detail": "Human-readable error message"
}
```

For validation errors (422):
```json
{
  "detail": [
    {
      "loc": ["body", "email"],
      "msg": "value is not a valid email address",
      "type": "value_error.email"
    }
  ]
}
```

### HTTP Status Codes

- `200` — Success (GET, sometimes POST/PATCH)
- `201` — Created (POST)
- `204` — No content (DELETE, sometimes POST)
- `400` — Bad request (client error in payload)
- `401` — Unauthenticated
- `403` — Forbidden (authenticated but not allowed)
- `404` — Not found
- `409` — Conflict (e.g., email already taken)
- `422` — Validation error
- `429` — Rate limit exceeded
- `500` — Server error
- `503` — Service unavailable (LLM/STT down)

---

## Endpoint Catalog

### Auth

#### `POST /api/auth/register`

Create a new user account.

Request:
```json
{
  "email": "user@example.com",
  "username": "budi_pku",
  "password": "secret123!",
  "display_name": "Budi P."  // optional
}
```

Response 201:
```json
{
  "user": {
    "id": 123,
    "email": "user@example.com",
    "username": "budi_pku",
    "display_name": "Budi P.",
    "xp": 0,
    "created_at": "2026-05-06T10:00:00Z"
  },
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."
}
```

Errors:
- `409` — Email or username already taken
- `422` — Invalid email/username/password format

Rules:
- Email: valid format, lowercased before storing
- Username: 3-32 chars, alphanumeric + underscore only
- Password: min 8 chars, must contain at least 1 letter and 1 number

---

#### `POST /api/auth/login`

Authenticate user and return tokens.

Request:
```json
{
  "email": "user@example.com",
  "password": "secret123!"
}
```

Response 200:
```json
{
  "user": { ... },
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."
}
```

Errors:
- `401` — Invalid credentials
- `429` — Too many failed attempts (5 per 15 min per IP)

---

#### `POST /api/auth/refresh`

Get new access token using refresh token.

Request:
```json
{
  "refresh_token": "eyJ..."
}
```

Response 200:
```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."
}
```

Errors:
- `401` — Invalid or expired refresh token

---

#### `POST /api/auth/logout`

Invalidate refresh token.

Headers: `Authorization: Bearer <access_token>`

Response 204 — No content

---

#### `GET /api/auth/me`

Get current user profile.

Headers: `Authorization: Bearer <access_token>`

Response 200:
```json
{
  "id": 123,
  "email": "user@example.com",
  "username": "budi_pku",
  "display_name": "Budi P.",
  "xp": 1250,
  "created_at": "2026-05-06T10:00:00Z",
  "last_login_at": "2026-05-06T15:30:00Z"
}
```

---

### Topics

#### `GET /api/topics`

List active topics.

Query params:
- `category` (optional): Filter by category
- `difficulty` (optional): `mudah`, `sedang`, `sulit`
- `limit` (optional, default 20, max 50)

Response 200:
```json
{
  "items": [
    {
      "id": "uuid",
      "motion": "Subsidi BBM harus dicabut total demi efisiensi anggaran",
      "category": "ekonomi",
      "difficulty": "sulit",
      "context": "Anggaran subsidi BBM Indonesia mencapai Rp 502 triliun di 2024..."
    }
  ],
  "total": 42
}
```

No pagination cursor for MVP. List is small.

---

#### `GET /api/topics/{id}`

Get a single topic.

Response 200:
```json
{
  "id": "uuid",
  "motion": "...",
  "category": "ekonomi",
  "difficulty": "sulit",
  "context": "..."
}
```

Errors:
- `404` — Topic not found

---

### Personas

#### `GET /api/personas`

List all 3 AI personas.

Response 200:
```json
{
  "items": [
    {
      "id": "uuid",
      "slug": "si_profesor",
      "name": "Si Profesor",
      "archetype": "Intelektual",
      "description": "Karakter intelektual yang sangat data-driven...",
      "tagline": "Data adalah senjata terbaik",
      "difficulty": "sulit",
      "target_score": 80
    },
    {
      "id": "uuid",
      "slug": "si_pak_rt",
      "name": "Si Pak RT",
      ...
    },
    {
      "id": "uuid",
      "slug": "si_aktivis",
      "name": "Si Aktivis 98",
      ...
    }
  ]
}
```

---

### Debates

#### `POST /api/debates`

Create a new debate session. Returns presigned upload URL.

Request:
```json
{
  "topic_id": "uuid",
  "persona_id": "uuid",
  "position": "PRO"  // or "KONTRA"
}
```

Response 201:
```json
{
  "id": "uuid",
  "topic": { ...full topic object... },
  "persona": { ...full persona object... },
  "position": "PRO",
  "status": "pending",
  "upload_url": "https://minio.example.com/audio-uploads/...?X-Amz-Signature=...",
  "upload_expires_at": "2026-05-06T16:05:00Z",  // 5 minutes from now
  "max_duration_sec": 60,
  "created_at": "2026-05-06T16:00:00Z"
}
```

Errors:
- `404` — Topic or persona not found
- `429` — Daily session limit reached (10/day for free tier)
- `422` — Invalid position value

Rules:
- User must be authenticated
- Daily limit: 10 sessions per user (configurable in env)
- Topic must be active (`is_active=true`)

---

#### `POST /api/debates/{id}/submit`

Notify backend that audio has been uploaded. Triggers async processing.

Headers: `Authorization: Bearer <access_token>`

Request:
```json
{}
```

Response 202:
```json
{
  "id": "uuid",
  "status": "transcribing",
  "estimated_completion_seconds": 30
}
```

Errors:
- `404` — Debate not found or not owned by user
- `400` — Debate already submitted or in invalid state
- `400` — Audio file not found in storage (upload failed)
- `400` — Audio file too large (>5MB) or too long (>65s)

---

#### `GET /api/debates/{id}/status`

Lightweight endpoint for polling. Returns current status only.

Headers: `Authorization: Bearer <access_token>`

Response 200:
```json
{
  "id": "uuid",
  "status": "scoring",  // pending, transcribing, scoring, generating_response, generating_clip, complete, failed
  "progress_percent": 45,  // rough estimate, optional
  "error_message": null
}
```

Errors:
- `404` — Debate not found or not owned by user

Recommended polling: every 2 seconds while not in terminal state.

---

#### `GET /api/debates/{id}`

Get full debate details. Use this once status is `complete`.

Headers: `Authorization: Bearer <access_token>`

Response 200:
```json
{
  "id": "uuid",
  "topic": { ... },
  "persona": { ... },
  "position": "PRO",
  "status": "complete",
  "transcript": "Justru karena beban anggaran terus membengkak, subsidi harus tepat sasaran...",
  "transcript_confidence": 0.97,
  "audio_duration_sec": 58.3,
  "opponent_response": "Pak/Bu, saya menghargai data Anda. Namun argumen 'tepat sasaran' selama ini gagal di lapangan karena...",
  "scores": {
    "logika": 8.5,
    "data": 9.0,
    "emosi": 6.5,
    "konsistensi": 8.8,
    "retorika": 7.2,
    "total": 78.4,
    "feedback_logika": "Argumen tersusun rapi dengan struktur claim-evidence-warrant.",
    "feedback_data": "Kutipan data BPS sangat relevan dan akurat.",
    "feedback_emosi": "Delivery agak monoton; coba variasikan intonasi pada poin kunci.",
    "feedback_konsistensi": "Posisi konsisten sepanjang argumen.",
    "feedback_retorika": "Penggunaan analogi dapat lebih vivid untuk meningkatkan dampak.",
    "feedback_overall": "Argumen kuat secara substantif. Tingkatkan delivery untuk memaksimalkan persuasi.",
    "result": "MENANG",
    "xp_earned": 180
  },
  "clip_url": "https://minio.example.com/clips/uuid.mp4",
  "created_at": "2026-05-06T16:00:00Z",
  "completed_at": "2026-05-06T16:00:45Z"
}
```

Errors:
- `404` — Debate not found or not owned by user

---

#### `GET /api/debates`

List user's debate history.

Headers: `Authorization: Bearer <access_token>`

Query params:
- `limit` (default 20, max 50)
- `cursor` (UUID of last item from previous page)
- `status` (optional filter: `complete`, `failed`)

Response 200:
```json
{
  "items": [
    {
      "id": "uuid",
      "topic": { "id": "uuid", "motion": "..." },
      "persona": { "id": "uuid", "name": "..." },
      "position": "PRO",
      "status": "complete",
      "scores": { "total": 78.4, "result": "MENANG" },
      "created_at": "2026-05-06T16:00:00Z"
    }
  ],
  "next_cursor": "uuid"  // null if no more
}
```

Note: Returns abbreviated debate objects. Use `GET /api/debates/{id}` for full details.

---

### User Stats

#### `GET /api/users/me/stats`

Aggregated stats for the current user.

Headers: `Authorization: Bearer <access_token>`

Response 200:
```json
{
  "total_debates": 23,
  "total_wins": 14,
  "total_losses": 8,
  "total_draws": 1,
  "win_rate": 0.61,
  "average_score": 72.4,
  "best_score": 91.0,
  "current_streak_days": 3,
  "favorite_persona": {
    "id": "uuid",
    "name": "Si Profesor",
    "matches": 12
  },
  "scores_by_dimension": {
    "logika": 7.8,
    "data": 7.2,
    "emosi": 6.4,
    "konsistensi": 8.0,
    "retorika": 6.9
  }
}
```

---

### Clips

#### `GET /api/clips/{debate_id}/share-meta`

Get metadata for sharing the clip on social media.

Response 200:
```json
{
  "clip_url": "https://...mp4",
  "thumbnail_url": "https://...jpg",
  "title": "Debat Politik: Subsidi BBM | Skor 78 di Arena Politika",
  "description": "Saya baru saja debat tentang Subsidi BBM dan dapat skor 78! Coba kamu...",
  "share_text_tiktok": "Skor debatku: 78/100 🔥 Coba juga di Arena Politika #ArenaPolitika #DebatPolitik",
  "share_text_instagram": "...",
  "share_text_whatsapp": "..."
}
```

---

## Rate Limiting

Implemented at two levels:

### Per-IP (Caddy + middleware)
- 30 requests/minute per IP for general endpoints
- 5 requests/15 minutes for `/api/auth/login` (anti brute-force)
- 3 requests/hour for `/api/auth/register` (anti spam)

### Per-User (FastAPI middleware + Redis)
- 10 debate sessions per day (free tier)
- 60 polling requests per minute (status checks)
- All other authenticated endpoints: 60 req/min

When rate limit exceeded, return:
```json
{
  "detail": "Rate limit exceeded. Try again in X seconds."
}
```
With header `Retry-After: <seconds>`.

---

## WebSocket (Optional, can be deferred)

For real-time progress updates instead of polling. Defer to v1.1 if HTTP polling is sufficient.

If implemented:
- Endpoint: `/ws/debates/{id}`
- Auth: Token in query string `?token=eyJ...`
- Server pushes status updates as JSON
- Client doesn't send messages; just listens
- Connection auto-closes when status is `complete` or `failed`

---

## Admin Endpoints (News Engine)

Restricted to user IDs in `ADMIN_USER_IDS` env var. Returns 403 for non-admins. Full spec in `specs/08_news_engine.md`.

#### `GET /api/admin/candidates`

List pending topic candidates from news engine.

Query params: `status` (default `pending_review`), `limit` (default 20)

Response 200:
```json
{
  "items": [
    {
      "id": "mongodb_object_id",
      "motion": "...",
      "category": "ekonomi",
      "difficulty": "sedang",
      "source_article_url": "https://...",
      "source_article_title": "...",
      "llm_model": "google/gemini-2.5-flash",
      "created_at": "2026-05-06T10:00:00Z"
    }
  ]
}
```

#### `POST /api/admin/candidates/{id}/approve`

Approve a candidate. Inserts into `topics` table with `source='news_engine'`.

Response 200: `{"ok": true, "topic_id": "uuid"}`

#### `POST /api/admin/candidates/{id}/reject`

Reject with reason.

Request: `{"reason": "Topic too narrow"}`

Response 200: `{"ok": true}`

---

## API Versioning Strategy

For MVP, no versioning. All endpoints under `/api/`.

For v2, add `/api/v2/` for new endpoints. Old `/api/` endpoints remain as v1, deprecated but functional for 6 months.
