# 09 — Development Roadmap

## Philosophy

13 weeks. Solo developer with AI assistance (Claude Code + Codex). Each week ends with a working, deployable increment.

If a week's tasks slip, push the next week. Don't add scope to "catch up" — that creates technical debt.

---

## Week 1: Foundation

**Goal:** Empty repo to "Hello World" running on VPS.

Tasks:
- [ ] Initialize git repo with this blueprint
- [ ] Set up backend folder: FastAPI scaffold, hello world endpoint
- [ ] Set up frontend folder: Vite + React + TS scaffold, hello world page
- [ ] Set up workers folder: Celery scaffold with one dummy task
- [ ] Write `infra/docker-compose.yml` (base) + Dockerfiles
- [ ] Configure Caddy with placeholder domain
- [ ] Provision VPS, run `server_init.sh`
- [ ] Deploy "hello world" to VPS, verify HTTPS works
- [ ] Set up CHANGELOG.md

**Definition of Done:**
- Visit `https://yourdomain.com` and see frontend
- Visit `https://yourdomain.com/api/health` and see `{"status": "ok"}`
- All containers running healthy

---

## Week 2: Database + Auth

**Goal:** User can register and login.

Tasks:
- [ ] Set up Alembic, create initial migration with `users` table
- [ ] Implement `app/auth/` module (router, service, schemas, models, JWT)
- [ ] Implement password hashing with Argon2
- [ ] Implement rate limiting (Redis-based)
- [ ] Frontend: login page, register page, auth state via Zustand
- [ ] Frontend: protected route wrapper
- [ ] Tests: auth endpoints, JWT generation/verification

**Definition of Done:**
- New user can register via UI
- User can login, get tokens, see protected page
- Rate limit kicks in on excessive login attempts
- All auth tests passing

---

## Week 3: Topics + Personas

**Goal:** Static reference data ready.

Tasks:
- [ ] Migration for `topics` and `personas` tables (with `source` column on topics)
- [ ] Implement `app/topics/` module
- [ ] Implement `app/personas/` module
- [ ] Write seed script (`backend/scripts/seed.py`)
- [ ] Owner writes 10-15 evergreen topics in `backend/scripts/evergreen_topics.py` covering 7 categories
- [ ] Seed 3 persona prompts from `prompts/opponent_prompts.md`
- [ ] Frontend: topic browser page
- [ ] Frontend: persona selector component
- [ ] Tests: topic and persona endpoints

**Definition of Done:**
- Seed runs successfully on fresh DB
- Logged-in user can browse 10-15 evergreen topics and select a persona
- Filtering by category and difficulty works

---

## Week 4: Audio Recording (Frontend)

**Goal:** User can record voice in browser.

Tasks:
- [ ] Implement `useAudioRecorder` hook using MediaRecorder API
- [ ] Implement `AudioRecorder` component with visual feedback
- [ ] Implement `Wavesurfer` integration for live waveform
- [ ] Handle mic permission flow gracefully
- [ ] Implement timer (counts down from 60s)
- [ ] Test on Chrome, Firefox, Safari (must work on at least these)
- [ ] Frontend: full debate creation flow (topic → persona → record)

**Definition of Done:**
- User can grant mic permission, record up to 60s, stop, replay
- Audio blob is valid WebM/Opus file
- Visual feedback works during recording
- Edge cases handled: permission denied, too short, browser crash

---

## Week 5: MinIO + Upload

**Goal:** Audio uploads to backend.

Tasks:
- [ ] Configure MinIO in docker-compose
- [ ] Initialize buckets via init script
- [ ] Implement `app/core/storage.py` for MinIO client
- [ ] Implement `app/debates/` module (just create + submit endpoints for now)
- [ ] Migration for `debates` table
- [ ] Implement presigned URL generation
- [ ] Frontend: upload audio to MinIO via presigned URL
- [ ] Frontend: notify backend after upload (`POST /debates/{id}/submit`)
- [ ] Audio metadata validation (FFprobe in worker)

**Definition of Done:**
- User completes recording, audio uploads to MinIO
- Backend validates and stores metadata
- Debate row exists in DB with status='transcribing'
- Audio file visible in MinIO console

---

## Week 6: Whisper STT Integration

**Goal:** Audio is transcribed.

Tasks:
- [ ] Implement `app/core/whisper.py` (OpenAI Whisper API client)
- [ ] Implement `workers/tasks/transcribe.py` Celery task
- [ ] Update debate status flow (transcribing → scoring)
- [ ] Implement local Whisper fallback (whisper-tiny)
- [ ] Implement transcript validation (length, language)
- [ ] Implement content moderation (LLM-based)
- [ ] Frontend: status polling endpoint, loading UI

**Definition of Done:**
- Submitted audio gets transcribed within 30s typically
- Indonesian audio produces accurate Indonesian transcript
- Invalid audio (silence, foreign language) is rejected gracefully
- Frontend shows progress: "Mentranskripsi suara..."

---

## Week 7: LLM Scoring

**Goal:** Transcripts produce 5-dimension scores.

Tasks:
- [ ] Implement `app/core/llm.py` (OpenRouter client with multi-model fallback)
- [ ] Implement `workers/tasks/score.py`
- [ ] Migration for `scores` table
- [ ] Tune scoring prompt (`prompts/scoring_prompt.md`) — iterate on real transcripts
- [ ] Implement score validation (clamp to 0-10, validate JSON)
- [ ] Implement weighted total calculation
- [ ] Implement win/lose/draw determination
- [ ] Implement XP calculation and award
- [ ] Test scoring on 20 real-ish transcripts (manually verify quality)

**Definition of Done:**
- Submitted transcript produces a structured score within 10s
- Scores feel "right" subjectively (test with owner reviews)
- XP increments correctly
- Score breakdown saved to `scores` table

---

## Week 8: Opponent Response + Result UI

**Goal:** AI generates counter-argument; user sees full result.

Tasks:
- [ ] Implement `workers/tasks/opponent.py`
- [ ] Tune persona prompts (`prompts/opponent_prompts.md`)
- [ ] Implement `GET /api/debates/{id}` full detail endpoint
- [ ] Frontend: result page with score breakdown
- [ ] Frontend: feedback display, opponent response display
- [ ] Frontend: animated reveal (skor counts up, etc.)
- [ ] Frontend: Cognitive Compass mini visualization (if implementing in MVP)

**Definition of Done:**
- After scoring completes, user sees beautiful result page
- All 5 dimensions visible with feedback
- AI opponent response visible
- Page is shareable (good OG meta tags)

---

## Week 9: Clip Generation

**Goal:** 15-25s highlight video is generated and downloadable.

Tasks:
- [ ] Design clip template (background image, font, watermark)
- [ ] Implement `workers/tasks/clip.py` with FFmpeg
- [ ] Implement segment selection (best 15-25s)
- [ ] Implement waveform visualization in clip
- [ ] Test clip generation on various transcript lengths
- [ ] Frontend: clip preview player
- [ ] Frontend: download button
- [ ] Implement audio cleanup task (delete after clip done)

**Definition of Done:**
- Clip generated within 15s of scoring complete
- Clip is 15-25s vertical MP4 with audio + visuals
- User can play and download clip
- Audio file deleted from MinIO after clip exists

---

## Week 10: Sharing + History

**Goal:** User can share clips and review past debates.

Tasks:
- [ ] Implement `GET /api/clips/{debate_id}/share-meta`
- [ ] Implement Open Graph meta tags for clip pages
- [ ] Frontend: share buttons (TikTok, Instagram, WhatsApp, Copy link)
- [ ] Frontend: history page (`GET /api/debates`)
- [ ] Frontend: stats page (`GET /api/users/me/stats`)
- [ ] Implement basic XP display in profile

**Definition of Done:**
- User can click share, gets native share dialog with clip
- History page shows all past debates
- Stats page shows aggregate metrics

---

## Week 11: News Engine

**Goal:** Auto-generate fresh debate topics from RSS news. See `specs/08_news_engine.md`.

Tasks:
- [ ] Add MongoDB to docker-compose
- [ ] Implement RSS poller (`workers/tasks/news/poll_rss.py`) for 5 sources
- [ ] Implement light scraper for full article fetch
- [ ] Implement LLM sanitization task
- [ ] Implement LLM motion generation task
- [ ] Implement banned-terms filter
- [ ] Implement quota target task (Beat-scheduled daily)
- [ ] Implement admin endpoints (list/approve/reject candidates)
- [ ] Tune `news_sanitization_prompt.md` and `motion_generation_prompt.md`
- [ ] Manual test: review 20 generated candidates, measure accuracy

**Definition of Done:**
- RSS polling runs hourly without errors
- 5-10 fresh candidates produced daily
- Owner can approve/reject via curl + admin endpoint
- Approved candidates appear in topics table with `source='news_engine'`
- LLM cost < Rp 5K/day for news engine

---

## Week 12: Polish + Testing

**Goal:** Fix bugs, improve UX, prepare for testing.

Tasks:
- [ ] Manual QA: walk through full user journey 10x
- [ ] Fix any bugs found
- [ ] Improve loading states and error messages
- [ ] Add retry mechanisms for failed debates
- [ ] Performance testing: load 50 concurrent users (use locust)
- [ ] Security review: check rate limits, validation, auth
- [ ] Set up backup script and verify restore
- [ ] Set up monitoring scripts (health check, disk check)
- [ ] Write user-facing FAQ / help page

**Definition of Done:**
- Full user journey works smoothly without bugs
- Backup verified working
- Monitoring alerts tested
- Documentation up to date

---

## Week 13: Alpha Launch

**Goal:** 30 alpha testers using the product.

Tasks:
- [ ] Create alpha tester onboarding flow (email invite, instructions)
- [ ] Set up feedback collection (Tally form or simple Google Form)
- [ ] Recruit 30 alpha testers from owner's network
- [ ] Launch! Monitor closely for first 72 hours
- [ ] Daily check-ins on metrics: signups, completions, errors, costs
- [ ] Weekly survey to alpha testers
- [ ] Identify top 3 issues and patch within week

**Definition of Done:**
- 30 testers signed up
- 100+ completed sessions in first 2 weeks
- Less than 5% session failure rate
- Top issues documented and prioritized for fixes

---

## After Week 13: Decision Point

Based on alpha results:

### Scenario A: Strong Engagement
- Day-7 retention >40%
- Avg sessions per user >3/week
- Subjective feedback positive
- → Proceed to v2 (party/election system)

### Scenario B: Mixed Engagement
- Some metrics good, others weak
- → Iterate on weakest area for 4 more weeks before deciding v2

### Scenario C: Weak Engagement
- Day-7 retention <20%
- → Honest pivot conversation with stakeholders

Don't build v2 in scenario C just because the blueprint exists. Ship value or kill the project.

---

## Out-of-Scope Tasks (Reminders)

These are NOT in the 12-week plan. Do not let scope creep:

- Mobile app (React Native or Flutter)
- Party/clan system
- Virtual elections
- Parliament/RUU system
- Cognitive Compass profiling
- Multiplayer (player vs player)
- Spectator mode
- Tournament system
- Payment/subscription
- Voiceprint anti-cheat
- TTS for AI opponent voice
- Indonesian dialect optimization
- Admin dashboard
- Analytics dashboard

If any of these comes up, note it for v2 planning. Do not build now.

---

## Estimated Effort Distribution

For solo developer with AI assistance:
- Backend code: 35%
- Frontend code: 30%
- Infrastructure/DevOps: 10%
- AI prompt engineering: 10%
- Testing: 8%
- Documentation: 5%
- Bug fixing/polish: 2%

For pair programming (you + Claude Code/Codex), expect AI to handle:
- ~70% of routine code generation
- ~40% of debugging
- ~10% of architectural decisions (you decide, AI helps articulate)

---

## Risk Register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Whisper API unreliable for Indonesian | Medium | High | Local fallback, tune prompts |
| LLM costs explode | Low | High | Daily limits, cost-based suspension |
| User finds way to abuse system | Medium | Medium | Manual review, ban system |
| Single VPS dies | Low | Critical | Daily backups, restore practiced |
| Solo dev burnout | High | Critical | Strict scope, weekly breaks |
| Scope creep from feature ideas | Very High | High | This blueprint, AGENTS.md, discipline |
| Performance issues at scale | Medium | Medium | Performance testing in week 11 |
| Content moderation false positives | Medium | Medium | Tunable thresholds, human review queue |

---

## Communication Cadence

- **Daily:** Update `CHANGELOG.md` with what was done
- **Weekly:** Self-review against this roadmap, adjust if needed
- **End of each week:** Demo to owner (yourself, in mirror) — does it work?
- **Mid-week 6 (halfway):** Honest review — are we on track?
- **End of week 13:** Alpha launch retrospective

---

## Tools and Productivity Tips

For maximum efficiency with AI assistance:

1. **Use Claude Code in agentic mode** for big features (week 5, 6, 7, 9). Let it generate full files, then review.
2. **Use Codex for utilities** — date helpers, audio segmentation, score formatting.
3. **Use Claude Code for refactoring** — it's better at understanding existing code.
4. **Don't fight the AI** — if it produces working code that meets the spec, accept it. Refactor later if patterns emerge.
5. **Review every commit** — AI mistakes happen. Trust but verify.
6. **Test as you go** — don't pile up untested code.
7. **Take breaks** — burnout will kill the project before scope will.

Good luck. Ship it.
