# Spec 08 — News Engine (Topic Auto-Generation)

## Purpose

Otomatis generate 5-10 topik debat baru per hari dari berita Indonesia, menggantikan kebutuhan seed manual.

## Pipeline

```
RSS Feeds + Scraping → Raw Articles (MongoDB)
         │
         ▼
    Dedup & Filter (kategori politik/ekonomi/sosial)
         │
         ▼
    LLM Sanitization (hapus nama tokoh/partai/institusi spesifik)
         │
         ▼
    LLM Motion Generation (ubah berita → motion debat)
         │
         ▼
    Editorial Queue (status='pending_review')
         │
         ▼
    Owner approve via simple admin endpoint
         │
         ▼
    Insert ke `topics` table (is_active=true)
```

## Sumber Berita

### RSS Feeds (priority 1)
- Kompas: `https://www.kompas.com/rss/`
- Tempo: `https://rss.tempo.co/`
- Detik: `https://rss.detik.com/`
- CNN Indonesia: `https://www.cnnindonesia.com/rss`
- Antara: `https://www.antaranews.com/rss`

Polling: setiap 1 jam via Celery Beat.

### Light Scraping (priority 2)
Untuk artikel yang RSS-nya tidak lengkap (hanya headline + summary), fetch full article:
- Library: `httpx` + `selectolax` (lebih cepat dari BeautifulSoup)
- Respect `robots.txt`
- User-agent jelas: `ArenaPolitika/1.0 (+https://arenapolitika.id/bot)`
- Rate limit: max 30 req/min per domain
- No JS rendering (jika perlu, skip)
- Cache full text di MongoDB (tidak fetch ulang)

## Storage

### MongoDB Collection: `news_articles`
```
{
  _id: ObjectId,
  source: "kompas" | "tempo" | "detik" | ...,
  url: string (unique index),
  title: string,
  content: string,
  category_raw: string (kategori asli sumber),
  published_at: datetime,
  fetched_at: datetime,
  status: "raw" | "filtered_in" | "filtered_out" | "processed"
}
```

### MongoDB Collection: `topic_candidates`
```
{
  _id: ObjectId,
  source_article_id: ObjectId (ref news_articles),
  motion: string,
  category: enum (7 kategori MVP),
  difficulty: enum (mudah/sedang/sulit),
  llm_model: string,
  llm_cost_usd: float,
  status: "pending_review" | "approved" | "rejected",
  created_at: datetime,
  reviewed_at: datetime | null,
  reviewed_by: int (user_id) | null,
  rejected_reason: string | null
}
```

Setelah approved → insert ke PostgreSQL `topics` table.

## Celery Tasks

```
workers/tasks/news/
├── poll_rss.py          # Per-jam, fetch RSS, dedup, save raw
├── scrape_article.py    # On-demand jika content kurang
├── filter_articles.py   # Tiap 6 jam, filter relevan (politik/ekonomi/sosial)
├── sanitize.py          # LLM call: hapus tokoh/partai/institusi spesifik
├── generate_motion.py   # LLM call: ubah berita → motion debatable
└── target_quota.py      # Tiap 24 jam, ensure ada 5-10 candidates baru
```

### Celery Beat Schedule
```python
beat_schedule = {
    'poll-rss-hourly': {'task': 'news.poll_rss', 'schedule': crontab(minute=0)},
    'filter-articles-6h': {'task': 'news.filter_articles', 'schedule': crontab(hour='*/6')},
    'generate-motions-daily': {'task': 'news.target_quota', 'schedule': crontab(hour=2, minute=0)},
}
```

## LLM Pipeline (2 calls per artikel)

### Call 1: Sanitization
- Model: Gemini 2.5 Flash
- Input: Full article text
- Output: Sanitized text (real names → generic)
- Max tokens: 1000
- Cost: ~Rp 5/artikel

### Call 2: Motion Generation
- Model: Gemini 2.5 Flash
- Input: Sanitized text
- Output: JSON `{motion, category, difficulty, rationale}`
- Max tokens: 300
- Cost: ~Rp 3/artikel

**Per topik baru: ~Rp 8 LLM cost.**
**Daily cost untuk 10 topik: ~Rp 80.** (negligible)

Prompt detail: `prompts/news_sanitization_prompt.md` dan `prompts/motion_generation_prompt.md` (file terpisah, di-tune setelah MVP live).

## Quota Logic

Daily quota: 5-10 motion candidates per hari.

```python
async def target_quota():
    pending_count = await count_candidates(status="pending_review")
    if pending_count >= 10:
        return  # Cukup, skip
    
    needed = 10 - pending_count
    articles = await get_unprocessed_articles(limit=needed * 2)  # 2x buffer
    
    for article in articles:
        sanitized = await sanitize(article)
        candidate = await generate_motion(sanitized)
        if candidate:
            await save_candidate(candidate)
```

## Editorial Review (Owner Workflow)

Untuk MVP, tidak ada admin UI. Pakai simple endpoint:

### Endpoints
```
GET  /api/admin/candidates              # List pending (auth: owner only)
POST /api/admin/candidates/{id}/approve # Approve → insert ke topics
POST /api/admin/candidates/{id}/reject  # Reject dengan reason
```

Auth: hardcode owner user_id di env `ADMIN_USER_IDS=1,2`. Jika user.id tidak di list → 403.

Owner buka URL admin sederhana (HTML page atau curl) sehari sekali, approve/reject batch. Estimasi: 5 menit/hari.

### Auto-Approval Threshold (Opsional, defer ke v2)
Jika LLM confidence tinggi + sumber kredibel + tidak ada banned terms, auto-approve. Untuk MVP: semua manual review.

## Banned Terms Check

Setelah motion di-generate, scan untuk:
- Nama partai (PDIP, Golkar, Gerindra, dll)
- Nama tokoh aktif top-30
- Nama institusi spesifik yang sensitif

Jika ditemukan → auto-reject candidate. Daftar di `app/news/banned_terms.py`, mudah di-update.

## Topics Mix Strategy

PostgreSQL `topics` table = mix dari:
1. **Auto-generated topics** (dari news engine) — fresh, current relevance
2. **Evergreen topics** (manual seed minimal, ~10-15 topik) — fallback jika news engine sepi

Frontend selector tidak membedakan keduanya.

Evergreen topics di-seed sekali saat first deploy. Tidak di-generate, tapi siap pakai untuk hari pertama sebelum news engine produktif. Owner tulis sendiri 10-15 topik timeless saat setup.

## Failure Modes

### RSS Feed Down
Skip source, lanjut yang lain. Log warning. Jika 3 hari berturut down, alert owner.

### LLM Cost Spike
Hard limit: max 50 LLM calls/day untuk news engine. Jika tercapai, stop sampai hari berikutnya.

### News Source Bermasalah Hukum
Jika ada complaint dari sumber, instant remove dari config. Tambahkan ke blocklist.

## Acceptance Criteria

- [ ] Cron polling RSS jalan tiap jam tanpa error
- [ ] MongoDB articles collection terisi dengan data dari minimal 3 sumber
- [ ] Sanitization menghapus 95%+ nama tokoh/partai (manual sampling 20 artikel)
- [ ] Motion generation produces valid JSON dengan kategori dan difficulty yang sesuai
- [ ] Daily quota 5-10 candidates terpenuhi
- [ ] Banned terms check berjalan, auto-reject kandidat bermasalah
- [ ] Admin endpoints accessible hanya untuk owner
- [ ] Approved candidates muncul di topics table dan visible di frontend selector
- [ ] LLM cost untuk news engine < Rp 5K/hari

## Out of Scope

- Admin UI (pakai endpoint + curl/Postman dulu)
- Auto-approval berdasarkan confidence
- Sentiment analysis
- Trending topic detection
- Multi-bahasa source (Indonesia only)
- Real-time push notification untuk topik trending
- News article archival/search untuk user
