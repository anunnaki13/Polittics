# Spec 05 — History & Player Stats

## Purpose

Allow players to review past debates and see aggregated stats about their performance. This drives retention by making progress visible.

## User Stories

### As a player
- I want to see a list of all my past debates
- I want to filter by status (selesai/gagal) and result (menang/kalah)
- I want to click on a past debate to see the full result again
- I want to see my overall stats: total debates, win rate, average score, best score
- I want to see which dimensions I'm strongest/weakest at
- I want to see my XP and any progression indicators

## Pages

### History Page (`/riwayat`)
List of all the user's debates, paginated.

Layout:
- Page title: "Riwayat Debat"
- Filter bar: All / Menang / Kalah / Seri / Gagal
- Sort: Terbaru / Skor Tertinggi
- List of debate cards

Each card shows:
- Topic motion (truncated)
- Persona name + difficulty
- Position (PRO/KONTRA)
- Date (relative: "2 hari lalu" or absolute for older)
- Score badge (large): "78" with color coding
- Result badge: "MENANG" / "KALAH" / "SERI"
- Click → navigate to `/riwayat/{debate_id}` (full detail page)

Pagination: cursor-based, 20 per page. "Muat lebih banyak" button at bottom.

### Debate Detail Page (`/riwayat/{id}`)
Same UI as the result page from `specs/02_debate_session.md`. Reuse components.

Difference: no "Debat Lagi" CTA at top (instead has "Kembali ke Riwayat" link).

### Stats Page (`/statistik`)
Aggregated performance metrics. Visual-heavy.

Sections:

**Overview cards:**
- Total Debat (number)
- Win Rate (%, with progress ring)
- Skor Rata-rata (number, /100)
- XP Total (number)

**Performance per dimension** (radar chart or 5 bars):
- Logika: 7.8
- Data: 7.2
- Emosi: 6.4
- Konsistensi: 8.0
- Retorika: 6.9

Highlights weakest dimension with text: "Tingkatkan: Emosi"

**Win/Loss distribution** (donut chart):
- Menang: 14 (61%)
- Kalah: 8 (35%)
- Seri: 1 (4%)

**Persona favorite:**
- "Lawan favorit: Si Profesor (12 debat, 67% menang)"

**Activity heatmap** (optional, defer to v2):
- 30-day calendar showing debate count per day

## API Endpoints

Already defined in `docs/05_api_design.md`:

- `GET /api/debates` — Paginated history list
- `GET /api/debates/{id}` — Single debate full detail
- `GET /api/users/me/stats` — Aggregated stats

### Implementation Notes for `/api/users/me/stats`

Computing stats on every request is fine for MVP scale (avg user has <100 debates). Use SQL aggregation:

```python
async def get_user_stats(db, user_id):
    stmt = (
        select(
            func.count(Debate.id).label("total_debates"),
            func.sum(case((Score.result == "MENANG", 1), else_=0)).label("wins"),
            func.sum(case((Score.result == "KALAH", 1), else_=0)).label("losses"),
            func.sum(case((Score.result == "SERI", 1), else_=0)).label("draws"),
            func.avg(Score.total).label("avg_score"),
            func.max(Score.total).label("best_score"),
            func.avg(Score.logika).label("avg_logika"),
            func.avg(Score.data).label("avg_data"),
            func.avg(Score.emosi).label("avg_emosi"),
            func.avg(Score.konsistensi).label("avg_konsistensi"),
            func.avg(Score.retorika).label("avg_retorika"),
        )
        .join(Score, Score.debate_id == Debate.id)
        .where(Debate.user_id == user_id, Debate.status == "complete")
    )
    result = await db.execute(stmt)
    row = result.one()
    
    # Find favorite persona (separate query)
    favorite = await get_favorite_persona(db, user_id)
    
    return UserStats(
        total_debates=row.total_debates,
        total_wins=row.wins or 0,
        total_losses=row.losses or 0,
        total_draws=row.draws or 0,
        win_rate=row.wins / row.total_debates if row.total_debates else 0,
        average_score=float(row.avg_score) if row.avg_score else 0,
        best_score=float(row.best_score) if row.best_score else 0,
        scores_by_dimension={
            "logika": float(row.avg_logika or 0),
            "data": float(row.avg_data or 0),
            "emosi": float(row.avg_emosi or 0),
            "konsistensi": float(row.avg_konsistensi or 0),
            "retorika": float(row.avg_retorika or 0),
        },
        favorite_persona=favorite,
    )
```

When user count grows large, cache result with 5-min TTL in Redis. Defer to v2.

## Frontend Implementation

### Files
```
frontend/src/
├── pages/
│   ├── HistoryPage.tsx
│   ├── DebateDetailPage.tsx
│   └── StatsPage.tsx
├── components/history/
│   ├── DebateCard.tsx
│   ├── HistoryFilters.tsx
│   └── EmptyHistory.tsx
└── components/stats/
    ├── OverviewCards.tsx
    ├── DimensionRadar.tsx
    └── WinLossDonut.tsx
```

### Empty State (No Debates Yet)
When user has 0 debates:

History page:
- Illustration (simple SVG)
- Text: "Belum ada riwayat debat"
- Subtitle: "Mulai debat pertamamu sekarang"
- CTA button: "Mulai Debat"

Stats page:
- Same illustration
- Text: "Statistik akan muncul setelah debat pertamamu"
- CTA button: "Mulai Debat"

### Loading State
Skeleton screens for cards. Don't show spinner — feels slow.

### Error State
"Tidak dapat memuat riwayat. Coba refresh halaman."

## XP & Progression (MVP-Light)

For MVP, XP is just a number that goes up. No leveling, no rewards, no progression bar (those are v2).

XP is awarded per completed debate based on:
- Result: MENANG = 100 base XP, SERI = 50, KALAH = 25
- Difficulty multiplier: mudah ×1.0, sedang ×1.3, sulit ×1.6
- Bonus for high score: +20% if score >85, +50% if score >95

```python
def calculate_xp(score_total: float, result: str, difficulty: str) -> int:
    base = {"MENANG": 100, "SERI": 50, "KALAH": 25}[result]
    multiplier = {"mudah": 1.0, "sedang": 1.3, "sulit": 1.6}[difficulty]
    bonus = 1.0
    if score_total > 95:
        bonus = 1.5
    elif score_total > 85:
        bonus = 1.2
    return int(base * multiplier * bonus)
```

Total XP shown in header (next to username) and stats page.

## Acceptance Criteria

- [ ] History page lists all completed debates, paginated
- [ ] Each debate card shows correct topic, persona, score, result
- [ ] Clicking a card navigates to detail page with full result
- [ ] Filters work correctly (by result)
- [ ] Sort works (Terbaru / Skor Tertinggi)
- [ ] Stats page shows accurate aggregate numbers
- [ ] Dimension averages computed correctly
- [ ] Favorite persona detected correctly
- [ ] Empty state shown when 0 debates
- [ ] Loading skeleton smooth and not janky
- [ ] All UI text in Bahasa Indonesia
- [ ] Page loads in <2s on average

## Performance Notes

For MVP, no caching needed:
- Average user has 5-50 debates total
- Stats query takes <50ms even without cache
- History query indexed on (user_id, created_at DESC)

If a user has 1000+ debates (unlikely in MVP), the queries will still be fast due to the index. Worry about that in v2.

## Out of Scope for MVP

- Public profile pages (other users seeing your stats)
- Leaderboards (global, weekly)
- Streaks (consecutive days)
- Achievements / badges
- Advanced filters (by topic, by date range)
- Charts beyond radar + donut (no historical trend lines)
- Export to CSV
- Comparison with other users
- Activity heatmap
- Detailed score progression over time
