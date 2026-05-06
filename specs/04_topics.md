# Spec 04 — Topics Management

## Purpose

Manage debate topics (motions). Topics berasal dari 2 sumber: news engine (auto-generated, lihat `08_news_engine.md`) dan evergreen seed (manual, ~10-15 topik fallback).

## Topic Sources

1. **Auto-generated** dari news engine — fresh, current, ~5-10 baru/hari setelah review
2. **Evergreen seed** — 10-15 topik manual saat first deploy, sebagai fallback hari pertama

Frontend tidak membedakan sumber. Semua topics rendered identik.

## Topic Data Model

Lihat `docs/04_database.md` untuk full schema. Recap:

```
topics
├── id: UUID
├── motion: TEXT
├── category: VARCHAR(32)  // 7 kategori
├── difficulty: VARCHAR(16)  // mudah/sedang/sulit
├── context: TEXT (nullable)
├── is_active: BOOLEAN
└── source: VARCHAR(32)  // 'news_engine' | 'evergreen_seed'
```

Tambahan kolom `source` di migration baru — untuk analytics dan debugging.

## Topic Quality Criteria

Setiap topik (manual atau auto-generated) wajib:

1. **Declarative statement**, bukan pertanyaan
2. **Debatable** — PRO dan KONTRA punya argumen substantif
3. **Bahasa Indonesia** formal tapi accessible
4. **Tanpa nama tokoh/partai/institusi spesifik**
5. **Tanpa SARA**
6. **Timeless atau slow-changing** (untuk evergreen)
7. Optional `context` 2-3 kalimat netral

Untuk auto-generated topics: criteria 1-5 di-enforce oleh prompt + banned terms check di news engine. Criteria 6 tidak relevan (justru dimaksudkan timely).

## 7 Categories

`ekonomi`, `sosial`, `hukum`, `lingkungan`, `pendidikan`, `kesehatan`, `politik`

## Difficulty Levels

| Level | Kriteria |
|---|---|
| Mudah | Posisi relatif jelas, argumen mudah dibangun dari common sense |
| Sedang | Butuh data/prinsip untuk dibahas |
| Sulit | Multifaset, butuh pengetahuan substantif |

Distribusi target di pool aktif: 30% mudah, 50% sedang, 20% sulit.

## Endpoints

Lihat `docs/05_api_design.md`:
- `GET /api/topics` — List dengan filter
- `GET /api/topics/{id}` — Single detail

Tidak ada POST/PATCH/DELETE publik. Topics dikelola via:
- Seed script (evergreen, sekali saat deploy)
- News engine + admin endpoints (auto-generated, ongoing)

## Frontend Topic Selector

UX:
- Page title: "Pilih Topik Debat"
- Filter: kategori dropdown, difficulty pills
- Grid topic cards (responsive)
- Card: motion (truncated 2 lines), kategori badge, difficulty stars
- Click → position picker

Sort default: random per session (variety). Jika frequently new topics datang dari news engine, urutan akan natural fresh.

## Topic Lifecycle

- Active: visible di selector
- Soft-disable: `is_active=false` (tidak muncul, tapi history tetap render)
- Tidak ada hard delete (preserve referential integrity dengan debates)

## Evergreen Seed (One-time at Deploy)

Owner tulis 10-15 topik timeless di `backend/scripts/evergreen_topics.py` saat first deploy. Tidak di-version control sebagai data — owner edit sendiri. Distribusi cover 7 kategori, mostly mudah-sedang.

Tujuan: hari pertama deployment ada topik tersedia sebelum news engine produksi.

## Acceptance Criteria

- [ ] News engine spec (08) implemented dan produksi 5-10 candidates/hari
- [ ] Evergreen seed berisi minimal 10 topik covering semua 7 kategori
- [ ] Frontend selector menampilkan campuran auto + evergreen tanpa beda visual
- [ ] Filter dan sort jalan
- [ ] Soft-disabled topics tidak muncul di selector tapi tetap di history

## Out of Scope for MVP

- User-submitted topic suggestions
- Trending topics ranking
- Topic voting
- Multi-bahasa
- Custom user topics
