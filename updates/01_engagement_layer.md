# Update 01 — Engagement Layer

> **Status:** Update spec for an existing running application. Application is already deployed on VPS with auth, debate session, scoring, clip generation, and history features functional.
>
> **Mandate:** This is an ADDITIVE update. Do NOT modify existing tables, endpoints, or business logic — only EXTEND. If anything in this spec conflicts with existing behavior, flag it and ask before changing.

---

## Context

The running app currently has player flow: register → debate → score → clip → share → repeat. XP is awarded but does nothing visible.

This update adds 4 retention mechanics:
1. League system (tiers + LP + promotion/demotion)
2. Streak tracking (consecutive days)
3. Daily missions (3 quests per day)
4. Adaptive difficulty + soft daily limits

Anti-cheat mechanics are **deferred** — not in this update.

---

## 1. League System

### Database Migration

New columns on existing `users` table:

```sql
ALTER TABLE users ADD COLUMN current_tier SMALLINT NOT NULL DEFAULT 7;
ALTER TABLE users ADD COLUMN lp INTEGER NOT NULL DEFAULT 0;
ALTER TABLE users ADD COLUMN promotion_wins SMALLINT NOT NULL DEFAULT 0;
ALTER TABLE users ADD COLUMN tier_changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
CREATE INDEX idx_users_tier_lp ON users (current_tier, lp DESC);
```

Tier numbering: lower number = higher rank. New users start at tier 7.

```
Tier 7: Akar Rumput      (start here)
Tier 6: Aktivis Kampung
Tier 5: Tokoh Lokal
Tier 4: Calon Legislatif
Tier 3: Anggota Dewan
Tier 2: Negarawan
Tier 1: Bapak Bangsa     (top)
```

Persist tier names in code (constant), not in DB. Tier numbers are stable contracts.

### LP Calculation

LP earned per debate is computed AFTER score determination. Replaces or supplements the existing `xp_earned` calculation — keep XP for compatibility but LP is the new primary currency.

```python
# Existing persona difficulty: mudah / sedang / sulit
# Map persona difficulty to "challenge level" relative to user tier:
#   - "matched": persona difficulty matches expected for user's tier
#   - "above": persona is harder than expected for tier
#   - "below": persona is easier than expected for tier

def expected_difficulty_for_tier(tier: int) -> str:
    if tier >= 6: return "mudah"    # Tier 7-6 face mostly Pak RT
    if tier >= 4: return "sedang"   # Tier 5-4 face mostly Aktivis
    return "sulit"                   # Tier 3-1 face mostly Profesor

def calculate_lp_change(result: str, persona_difficulty: str, user_tier: int) -> int:
    expected = expected_difficulty_for_tier(user_tier)
    levels = {"mudah": 1, "sedang": 2, "sulit": 3}
    relative = levels[persona_difficulty] - levels[expected]
    # relative: -2..-1 = below, 0 = matched, +1..+2 = above

    if result == "MENANG":
        if relative >= 1: return 40
        if relative == 0: return 20
        return 10  # Easy win, less LP
    elif result == "KALAH":
        if relative >= 1: return 0   # Lost to harder opponent, no penalty
        if relative == 0: return -15
        return -25  # Lost to easier opponent, big penalty
    else:  # SERI
        return 5
```

### Promotion Mechanic

When `lp >= 100`, user enters **promotion track**:
- Next 3 debates are "promotion matches"
- Must win 2 out of 3 to promote (tier - 1, lp reset to 50, promotion_wins reset to 0)
- If fail (lose 2 of 3), demote back to lp=70, promotion_wins reset

Implementation logic in service layer:

```python
async def apply_lp_change(user_id, lp_delta, result):
    user = await get_user(user_id)

    # In promotion mode if lp was >= 100 at debate start
    in_promotion = user.lp >= 100

    if in_promotion:
        if result == "MENANG":
            user.promotion_wins += 1
            if user.promotion_wins >= 2:
                # PROMOTE
                user.current_tier = max(1, user.current_tier - 1)
                user.lp = 50
                user.promotion_wins = 0
                user.tier_changed_at = utcnow()
                await emit_event("tier_promoted", user_id, new_tier=user.current_tier)
        else:
            user.promotion_wins -= 1 if result == "KALAH" else 0
            if user.promotion_wins <= -2:
                # FAIL PROMOTION
                user.lp = 70
                user.promotion_wins = 0
    else:
        # Normal LP change
        new_lp = user.lp + lp_delta
        if new_lp <= 0 and user.current_tier < 7:
            # DEMOTE
            user.current_tier += 1
            user.lp = 50  # Land at middle of new tier
            user.tier_changed_at = utcnow()
            await emit_event("tier_demoted", user_id, new_tier=user.current_tier)
        else:
            user.lp = max(0, new_lp)

    await db.commit()
```

### API Endpoints

Add to existing `users` module:

```
GET /api/users/me/league
```

Response 200:
```json
{
  "current_tier": 5,
  "current_tier_name": "Tokoh Lokal",
  "lp": 78,
  "in_promotion": false,
  "promotion_wins": 0,
  "promotion_target_tier": 4,
  "promotion_target_tier_name": "Calon Legislatif",
  "lp_to_promotion": 22
}
```

### Frontend Changes

**Header bar:** Add tier badge + LP progress bar next to existing XP display. Keep XP visible but smaller.

**After-debate result page:** Add LP change animation. Show:
- Old LP → New LP with animated counter
- "+20 LP" or "-15 LP" floating label
- If promotion triggered: special modal with celebration animation
- If demotion triggered: muted notification, no celebration

**New page:** `/league` — shows tier ladder visualization, current position, recent LP history (last 20 debates), upcoming promotion match indicator.

---

## 2. Streak System

### Database

New table:

```sql
CREATE TABLE user_streaks (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    current_streak INTEGER NOT NULL DEFAULT 0,
    longest_streak INTEGER NOT NULL DEFAULT 0,
    last_active_date DATE NOT NULL,
    streak_started_at DATE NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Streak Logic

Streak increments when user completes at least 1 debate per day (UTC date).

```python
async def update_streak_on_debate_complete(user_id):
    today_utc = date.today()
    streak = await get_or_create_streak(user_id, today_utc)

    if streak.last_active_date == today_utc:
        # Already counted for today, no change
        return streak

    days_diff = (today_utc - streak.last_active_date).days

    if days_diff == 1:
        # Consecutive day, increment
        streak.current_streak += 1
        streak.longest_streak = max(streak.longest_streak, streak.current_streak)
    elif days_diff > 1:
        # Broke streak
        streak.current_streak = 1
        streak.streak_started_at = today_utc
    # days_diff < 0 should never happen (clock issue)

    streak.last_active_date = today_utc
    await db.commit()
    return streak
```

This must run inside the same transaction as debate completion. Add to existing `cleanup_audio` task or wherever `status='complete'` is set.

### LP Bonus Tiers

Apply as multiplier on LP earned:

```python
def streak_lp_multiplier(streak_days: int) -> float:
    if streak_days < 4: return 1.10    # +10%
    if streak_days < 8: return 1.20    # +20%
    if streak_days < 15: return 1.30   # +30%
    return 1.50                         # +50% for 15+ day streaks
```

Multiplier applies only on POSITIVE LP changes. Negative LP not amplified.

### API Endpoint

```
GET /api/users/me/streak
```

Response 200:
```json
{
  "current_streak": 7,
  "longest_streak": 14,
  "last_active_date": "2026-05-06",
  "next_milestone_days": 1,
  "next_milestone_bonus": "30%",
  "active_today": true,
  "active_yesterday": true
}
```

### Frontend

**Header:** Small flame icon + streak number when active. Tap → `/streak` detail page.

**Daily reminder:** If user logged in today but hasn't started a debate, show subtle banner: "Streak {n} hari — main 1 debat untuk lanjutkan."

**Streak break warning:** If user is online and clock approaching midnight UTC without playing today, show warning toast.

---

## 3. Daily Missions

### Database

```sql
CREATE TABLE daily_missions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    mission_date DATE NOT NULL,
    mission_type VARCHAR(64) NOT NULL,
    target_value INTEGER NOT NULL,
    current_progress INTEGER NOT NULL DEFAULT 0,
    lp_reward INTEGER NOT NULL,
    completed_at TIMESTAMPTZ NULL,
    claimed_at TIMESTAMPTZ NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_user_mission_unique ON daily_missions (user_id, mission_date, mission_type);
CREATE INDEX idx_missions_user_date ON daily_missions (user_id, mission_date DESC);
```

### Mission Types (MVP set — 8 types)

```python
MISSION_TYPES = {
    "win_count": {
        "label_template": "Menangkan {target} debat hari ini",
        "target_range": (1, 3),
        "lp_per_target": 20,
    },
    "win_vs_difficulty": {
        "label_template": "Menangkan 1 debat melawan persona {difficulty}",
        "target_range": (1, 1),
        "lp_per_target": 50,
        "metadata": {"difficulty": ["sedang", "sulit"]},  # randomized
    },
    "min_score": {
        "label_template": "Dapat skor total minimal {target}",
        "target_range": (70, 85),
        "lp_per_target": 0.5,  # 35-42 LP
    },
    "min_dimension_score": {
        "label_template": "Dapat skor {dimension} minimal {target}",
        "target_range": (7, 9),
        "lp_per_target": 5,
        "metadata": {"dimension": ["logika", "data", "emosi", "konsistensi", "retorika"]},
    },
    "complete_count": {
        "label_template": "Selesaikan {target} debat hari ini",
        "target_range": (3, 5),
        "lp_per_target": 12,
    },
    "try_new_category": {
        "label_template": "Coba kategori {category} (belum pernah hari ini)",
        "target_range": (1, 1),
        "lp_per_target": 30,
    },
    "share_clip": {
        "label_template": "Bagikan {target} klip",
        "target_range": (1, 2),
        "lp_per_target": 15,
    },
    "perfect_dimension": {
        "label_template": "Dapat skor {dimension} 9.0+ dalam satu debat",
        "target_range": (1, 1),
        "lp_per_target": 60,
        "metadata": {"dimension": ["logika", "data", "emosi", "konsistensi", "retorika"]},
    },
}
```

### Daily Generation

Celery Beat scheduled task at 00:05 UTC (just after midnight):

```python
@celery_app.task
async def generate_daily_missions():
    """Generate 3 random missions per active user for today."""
    today = date.today()
    active_user_ids = await get_recently_active_users(days=14)

    for user_id in active_user_ids:
        existing = await count_missions(user_id, today)
        if existing >= 3:
            continue

        # Pick 3 distinct mission types weighted toward variety
        chosen_types = sample_mission_types(count=3)
        for mission_type in chosen_types:
            mission = build_mission(user_id, today, mission_type)
            await save_mission(mission)
```

Lazy generation alternative: when user first hits `/api/missions/today` and it's empty, generate inline. Use this if Celery Beat is overloaded.

### Progress Tracking

Hook into existing debate completion flow. After a debate transitions to `status='complete'`:

```python
async def update_missions_on_debate_complete(user_id, debate, scores):
    today = date.today()
    missions = await get_active_missions(user_id, today)

    for mission in missions:
        if mission.completed_at is not None:
            continue

        progress_delta = compute_progress_delta(mission, debate, scores)
        if progress_delta > 0:
            mission.current_progress += progress_delta
            if mission.current_progress >= mission.target_value:
                mission.completed_at = utcnow()
                # LP awarded on claim, not auto

    await db.commit()
```

### API Endpoints

```
GET  /api/missions/today
POST /api/missions/{id}/claim
```

`GET /api/missions/today` Response 200:
```json
{
  "date": "2026-05-06",
  "missions": [
    {
      "id": "uuid",
      "type": "win_count",
      "label": "Menangkan 2 debat hari ini",
      "target_value": 2,
      "current_progress": 1,
      "lp_reward": 40,
      "completed": false,
      "claimed": false
    }
  ]
}
```

`POST /api/missions/{id}/claim`:
- Returns 400 if not yet completed or already claimed
- Awards LP (with streak multiplier applied)
- Sets `claimed_at = NOW()`

### Frontend

**New widget on home page:** "Misi Harian" — 3 cards showing progress bars. Completed missions get a "Klaim" button. Claimed missions show LP reward earned.

**Reset behavior:** Old missions auto-disappear at midnight UTC (filtered by `mission_date == today`). Claimable missions remain claimable for 24 hours; unclaimed completed missions expire (LP forfeit). This creates urgency.

---

## 4. Adaptive Difficulty Suggestion

### Logic

Track win/loss streak in last 5 debates. Recommend persona based on pattern:

```python
async def get_recommended_persona(user_id) -> Persona:
    recent = await get_last_n_debates(user_id, n=5)

    if not recent:
        return await get_persona_by_slug("si_pak_rt")  # New users start easy

    last_3_results = [d.scores.result for d in recent[:3]]
    win_streak = len([r for r in last_3_results if r == "MENANG"])
    loss_streak = len([r for r in last_3_results if r == "KALAH"])

    user = await get_user(user_id)
    expected = expected_difficulty_for_tier(user.current_tier)

    if loss_streak >= 3:
        # Suggest 1 step easier
        return get_easier_persona(expected)
    if win_streak >= 3:
        # Suggest 1 step harder
        return get_harder_persona(expected)

    return get_persona_by_difficulty(expected)
```

### API Endpoint

Augment existing `GET /api/personas` response with optional recommendation:

```
GET /api/personas?include_recommendation=true
```

Response 200:
```json
{
  "items": [...],
  "recommendation": {
    "persona_id": "uuid",
    "reason": "win_streak_3"  // or "loss_streak_3", "tier_match", "new_user"
  }
}
```

Frontend uses the recommendation to highlight one persona card with "Disarankan untukmu" badge. User can still pick any persona — this is suggestion, not enforcement.

---

## 5. Soft Daily Limit Warnings

### Logic

Existing hard limit (10 sessions/day) stays. Add soft warnings:

```python
async def check_session_warning(user_id) -> Optional[str]:
    today_count = await get_today_session_count(user_id)

    if today_count == 5:
        return "session_5_pacing"
    if today_count == 8:
        return "session_8_fatigue"
    if today_count == 10:
        return "session_10_limit"

    return None
```

### API Behavior

When user calls `POST /api/debates` (create new debate), include warning in response if applicable:

```json
{
  "id": "uuid",
  "topic": {...},
  "persona": {...},
  "upload_url": "...",
  "warning": {
    "type": "session_5_pacing",
    "message": "Sudah 5 debat hari ini! Skor cenderung menurun saat lelah. Pertimbangkan istirahat sejenak."
  }
}
```

Frontend displays warning as non-blocking toast or modal that user must dismiss before recording.

### Cooldown Suggestion

After 8 sessions, cap further sessions to 1 per 30 minutes. Implement as Redis rate limit specific to "high-volume users":

```python
if today_count >= 8:
    cooldown_key = f"cooldown:debate:{user_id}"
    if await redis.exists(cooldown_key):
        ttl = await redis.ttl(cooldown_key)
        raise RateLimitError(f"Istirahat {ttl//60} menit lagi sebelum debat berikutnya")
    await redis.setex(cooldown_key, 1800, "1")  # 30 min
```

This is intentionally "annoying" for healthy retention. Player will return tomorrow fresh instead of grinding.

---

## Migration Order

Run migrations in this order on existing DB:

```
1. ALTER users (add tier, lp, promotion_wins, tier_changed_at)
2. CREATE TABLE user_streaks
3. CREATE TABLE daily_missions
4. Backfill: set all existing users to tier=7, lp=0
5. Backfill: create user_streaks rows from existing user.created_at
```

Backfill SQL example:

```sql
INSERT INTO user_streaks (user_id, current_streak, longest_streak, last_active_date, streak_started_at)
SELECT id, 0, 0, CURRENT_DATE, CURRENT_DATE
FROM users
ON CONFLICT (user_id) DO NOTHING;
```

For users who have already played before this update: their LP starts at 0 regardless of past XP. This is intentional — clean slate for fairness.

---

## Backend Module Structure

Add new modules:

```
backend/app/
├── league/
│   ├── __init__.py
│   ├── service.py        # LP calculation, promotion logic
│   ├── tiers.py          # Tier names, expected difficulties
│   ├── router.py         # GET /api/users/me/league
│   └── schemas.py
├── streaks/
│   ├── __init__.py
│   ├── service.py
│   ├── router.py         # GET /api/users/me/streak
│   └── schemas.py
└── missions/
    ├── __init__.py
    ├── generator.py      # Mission generation logic
    ├── service.py        # Progress tracking
    ├── router.py         # GET /api/missions/today, POST claim
    └── schemas.py

workers/tasks/
└── missions/
    └── generate.py       # Celery Beat task
```

Hook into existing debate completion:

```python
# In existing workers/tasks/cleanup.py or wherever debate.status='complete' is set
async def on_debate_complete(debate_id):
    debate = await get_debate(debate_id)

    # NEW: League LP update
    lp_delta = calculate_lp_change(debate.scores.result, debate.persona.difficulty, debate.user.current_tier)
    streak_multiplier = await get_streak_multiplier(debate.user_id)
    final_lp = int(lp_delta * streak_multiplier) if lp_delta > 0 else lp_delta
    await apply_lp_change(debate.user_id, final_lp, debate.scores.result)

    # NEW: Streak update
    await update_streak_on_debate_complete(debate.user_id)

    # NEW: Mission progress
    await update_missions_on_debate_complete(debate.user_id, debate, debate.scores)

    # Existing logic continues...
```

---

## Testing Strategy

### Unit tests
- LP calculation matrix (5 results × 3 difficulties × 7 tiers)
- Promotion / demotion edge cases
- Streak increment, break, restart
- Mission progress for each mission type

### Integration tests
- Full debate flow: create → submit → complete → verify LP, streak, missions all updated
- Promotion match flow: 100 LP → win 2 of 3 → tier change
- Demotion flow: 0 LP at non-min tier → tier increase

### Manual smoke tests
- Player at tier 7 plays 5 wins in a row → reaches promotion → completes promotion → tier 6
- Player at tier 5 takes 3 days break → returns → streak = 1 (not continued)
- Player completes daily mission → LP awarded with streak multiplier applied

---

## Acceptance Criteria

- [ ] Migration runs cleanly on existing production DB without breaking existing data
- [ ] Existing users see tier 7 + LP 0 after migration
- [ ] After completing a debate, LP changes match formula
- [ ] Promotion match modal appears at LP=100, fires correctly after 2 wins
- [ ] Streak increments correctly across UTC day boundaries
- [ ] 3 missions generated per active user per day
- [ ] Mission progress updates within 5s of debate completion
- [ ] Mission claim awards LP with streak multiplier applied
- [ ] Adaptive recommendation suggests easier persona after 3 losses
- [ ] Soft warnings appear at 5 and 8 sessions without blocking
- [ ] All UI text in Bahasa Indonesia
- [ ] Existing debate flow not broken (smoke test full happy path)

---

## Out of Scope for This Update

These are mentioned in the broader gamification design but NOT in this update:

- Hidden bonus / variable reward system (separate update)
- Mentor whisper / replay analysis (separate update)
- Multi-layer leaderboards (separate update)
- Weekly controversy event (separate update)
- Quotable feed and counter mechanic (separate update, v1.5)
- Anti-cheat layer (deferred per owner instruction)
- Social/friends system (v2)
- Comeback mechanic for inactive users (post-alpha)

Each will get its own `updates/NN_*.md` file when the time comes.

---

## Rollout Plan

Recommended sequence:

1. **Day 1:** Deploy DB migrations (with feature flags off)
2. **Day 2:** Deploy backend code (endpoints respond but frontend not yet using)
3. **Day 3:** Test with owner account only — full flow
4. **Day 4:** Deploy frontend with feature flag — owner enables for self
5. **Day 5-7:** Monitor logs, fix bugs, verify metrics
6. **Day 8:** Enable feature flag for all users
7. **Day 9-14:** Watch retention metrics, compare to pre-update baseline

If metrics show negative impact (lower D1 retention, more bounces), feature-flag off and investigate. Feature flag pattern:

```python
ENGAGEMENT_LAYER_ENABLED = settings.FEATURE_ENGAGEMENT_LAYER  # env var
```

Wrap all new logic with this flag. Default to off in `.env.example`.
