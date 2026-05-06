# Arena Politika — MVP Blueprint

> **Voice-based political debate game for Indonesian audience.**
> Status: Pre-development. This repo contains specifications only.

---

## ⚡ Quick Context for AI Agents (Claude Code / Codex)

**You are reading the source-of-truth specification for the Arena Politika MVP.**

Before generating any code:

1. Read `AGENTS.md` for working agreement and conventions.
2. Read `docs/01_overview.md` for product context.
3. Read `docs/02_architecture.md` for technical architecture.
4. Read the relevant `specs/` file for the feature you are implementing.
5. Follow `docs/03_conventions.md` for code style and patterns.

**This MVP is intentionally narrow in scope.** Do not implement features outside the MVP scope (see `docs/01_overview.md`). Suggest deferred features for v2 only when explicitly asked.

---

## 🎯 MVP Scope (One Sentence)

A web app where a player records a 60-second voice argument on a political topic, gets scored by AI on 5 dimensions, sees a transcript and feedback, and can share an auto-generated highlight clip.

**That's it.** No parties, no elections, no parliament, no career. Those are v2+.

---

## 🛠 Technology Stack

| Layer | Choice | Why |
|---|---|---|
| Backend | FastAPI (Python 3.11+) | Familiar to team, async-friendly, fast |
| Database | PostgreSQL 16 | Relational data, mature |
| Cache/Queue | Redis 7 | Sessions, rate limiting, Celery broker |
| Object Storage | MinIO (self-hosted) | S3-compatible, runs on same VPS |
| Background Worker | Celery | Audio processing, scoring, clip generation |
| Frontend | React 18 + Vite + TypeScript | Standard, fast iteration |
| Styling | TailwindCSS + Shadcn/ui | Quick, consistent |
| State | Zustand + TanStack Query | Lightweight |
| Audio Viz | Wavesurfer.js | Audio waveform display |
| STT | OpenAI Whisper API | Best accuracy/cost ratio for MVP |
| LLM Gateway | OpenRouter | Multi-model fallback |
| Primary LLM | Gemini 2.5 Flash | Cheap, fast, good Indonesian |
| Premium LLM | Claude Haiku 4.5 | Higher quality scoring |
| Auth | Self-hosted JWT | Simple email+password for MVP |
| Payment | None for MVP | Add Midtrans later |
| Reverse Proxy | Caddy | Auto-HTTPS, simpler than Nginx |
| Container | Docker + Docker Compose | Standard |
| Hosting | Single VPS (Hetzner/Contabo) | Sufficient for MVP |

---

## 📂 Repository Structure (To Be Created)

```
arena-politika/
├── backend/           # FastAPI application
├── frontend/          # React + Vite application
├── workers/           # Celery workers
├── infra/             # Docker compose, Caddy config
├── scripts/           # Deployment, migration scripts
├── docs/              # This blueprint (read-only reference)
└── specs/             # Feature specifications (read-only reference)
```

---

## 🗺 Documentation Map

### Core Documentation (`docs/`)
- **[01_overview.md](docs/01_overview.md)** — Product overview, MVP scope, user journey
- **[02_architecture.md](docs/02_architecture.md)** — System architecture, data flow, deployment
- **[03_conventions.md](docs/03_conventions.md)** — Coding conventions, project structure, git workflow
- **[04_database.md](docs/04_database.md)** — Database schema, migrations
- **[05_api_design.md](docs/05_api_design.md)** — API endpoints, request/response shapes
- **[06_deployment.md](docs/06_deployment.md)** — VPS setup, Docker compose, deployment flow
- **[07_ai_pipeline.md](docs/07_ai_pipeline.md)** — STT + LLM scoring pipeline detail
- **[08_security.md](docs/08_security.md)** — Auth, rate limiting, input validation
- **[09_roadmap.md](docs/09_roadmap.md)** — 12-week development roadmap

### Feature Specifications (`specs/`)
- **[01_auth.md](specs/01_auth.md)** — User registration, login, JWT
- **[02_debate_session.md](specs/02_debate_session.md)** — Core gameplay: voice recording → scoring
- **[03_ai_personas.md](specs/03_ai_personas.md)** — 3 AI persona prompts and behavior
- **[04_topics.md](specs/04_topics.md)** — Topic management (manual seeding for MVP)
- **[05_history.md](specs/05_history.md)** — Player debate history and stats
- **[06_clip_generator.md](specs/06_clip_generator.md)** — Auto-generated highlight clip
- **[07_share.md](specs/07_share.md)** — Share clip to social media
- **[08_news_engine.md](specs/08_news_engine.md)** — Auto topic generation from news (RSS + LLM)

### AI Agent Prompts (`prompts/`)
- **[scoring_prompt.md](prompts/scoring_prompt.md)** — System prompt for AI Judge scoring
- **[opponent_prompts.md](prompts/opponent_prompts.md)** — System prompts for 3 personas

---

## 🚀 Development Workflow with AI Agents

### For Claude Code

1. Open this repo, read `AGENTS.md` first.
2. Pick a task from `docs/09_roadmap.md` (in order).
3. Read the relevant `specs/*.md` for that task.
4. Implement following `docs/03_conventions.md`.
5. Write tests as you go (see Testing section in `03_conventions.md`).
6. Commit with conventional commit format (e.g., `feat(auth): add jwt middleware`).

### For Codex GPT

1. Same as above. Codex should read `AGENTS.md` and follow the same conventions.
2. Use Codex for: complex algorithms (scoring math), data transformation utilities, frontend animations.
3. Use Claude Code for: feature implementation, integration, refactoring.

### Coordination Between Agents

Both agents should:
- Update `CHANGELOG.md` after each merged feature.
- Never modify files in `docs/` or `specs/` (those are specs, not code).
- Always run tests before committing.
- Follow naming conventions strictly (see `03_conventions.md`).

---

## 🎯 MVP Success Criteria

The MVP is considered complete when:

- [ ] User can register, login, logout
- [ ] User can start a debate session by selecting a topic + AI persona
- [ ] User can record voice (up to 60 seconds) in browser
- [ ] System transcribes voice using Whisper
- [ ] AI scores the argument on 5 dimensions (Logika, Data, Emosi, Konsistensi, Retorika)
- [ ] User sees scored result with text feedback
- [ ] User can view their debate history
- [ ] System generates a 15-25 second highlight video clip
- [ ] User can download or share the clip
- [ ] Application runs on single VPS via Docker Compose
- [ ] Total infrastructure cost under Rp 6 million/month at 200 sessions/day

---

## 🚫 Out of MVP Scope (Do Not Build)

The following are **explicitly excluded** from MVP. Mention them only if explicitly asked:

- ❌ Political party / clan system
- ❌ Virtual elections
- ❌ Parliament / RUU voting
- ❌ Career progression (just basic XP for MVP)
- ❌ Cognitive Compass profile
- ❌ Multiplayer debate (player vs player)
- ❌ Spectator mode
- ❌ Tournament system
- ❌ Mobile app (web-only for MVP)
- ❌ Payment/subscription
- ❌ Anti-cheat (basic only — voiceprint deferred)

---

## 📞 Contact

Project Owner: Budi (CV Panda Global Teknologi)
Repository: TBD
License: Proprietary (private)
