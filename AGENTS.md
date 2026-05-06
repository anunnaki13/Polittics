# AGENTS.md — Working Agreement for AI Coding Agents

> **This file defines how Claude Code and Codex GPT should behave when working on this codebase.**

---

## 🎯 Mission

You are helping build the MVP of **Arena Politika** — a voice-based political debate game. The product owner is Budi from CV Panda Global Teknologi.

## 🧭 First Principles

1. **MVP First, Always.** If a task is not in `docs/09_roadmap.md` or a `specs/*.md` file, do not build it. Suggest deferring instead.
2. **Spec-Driven.** Read the relevant spec before implementing. Specs override your assumptions.
3. **Working Software Over Perfection.** Ship the simplest thing that meets the spec. Refactor later.
4. **No Surprises.** When in doubt, ask. Do not invent requirements, libraries, or APIs.
5. **Indonesian Context.** UI text is Bahasa Indonesia. Code, comments, and commit messages are English.

---

## 📁 File Boundaries

### You MAY freely modify:
- `backend/**` (except `backend/migrations/` after they are committed)
- `frontend/**`
- `workers/**`
- `infra/**`
- `scripts/**`
- `tests/**`
- `CHANGELOG.md`
- `README.md` (only the "Repository Structure" and "Status" sections)

### You MUST NOT modify without explicit instruction:
- `docs/**` — These are specifications. They are the source of truth.
- `specs/**` — Feature specifications. Same.
- `prompts/**` — AI prompts that have been tuned. Changing them affects scoring quality.
- `AGENTS.md` — This file.
- `.env.example` — Only add, do not remove keys.
- Existing migrations once committed.

If you believe a doc/spec is wrong, **flag it in your response** and ask the owner to update it. Do not silently change it.

---

## 🗺 Task Workflow

### Step 1: Understand
- Read `docs/09_roadmap.md` to see current week and tasks
- Read the relevant `specs/*.md` for the task
- Check `CHANGELOG.md` for what is already done

### Step 2: Plan
- Outline implementation in 3-5 bullet points
- Identify any ambiguity in the spec — ask before building
- Estimate complexity (S/M/L). If L, propose breaking it down.

### Step 3: Implement
- Follow `docs/03_conventions.md`
- Write code in small, reviewable chunks
- Add tests for new logic (see Testing below)

### Step 4: Verify
- Run linter, type checker, tests
- Test the happy path manually if it's user-facing
- Confirm no regressions

### Step 5: Document
- Update `CHANGELOG.md` with a one-line entry under "Unreleased"
- Update API docs if endpoints changed
- Add inline comments only for non-obvious logic

---

## 🛡 Non-Negotiables

### Security
- **Never commit secrets.** Use `.env` files and reference `.env.example`.
- All user input must be validated (Pydantic on backend, Zod on frontend).
- All endpoints except auth/public ones require JWT.
- Use parameterized queries always (SQLAlchemy ORM or asyncpg with `$1` style).
- Hash passwords with `argon2` (not bcrypt, not plain MD5/SHA).

### Privacy
- Voice recordings are deleted after transcription + scoring is complete (within 24h).
- Transcripts are kept (they are training data and game history).
- User cannot search for other users' debates without their consent.
- IP addresses are stored only for rate limiting, hashed.

### Cost Discipline
- LLM calls are expensive. Always:
  - Use Gemini Flash as default model
  - Cache where possible
  - Set `max_tokens` conservatively
  - Implement rate limiting per user (10 sessions/day for free tier)
- STT calls are expensive. Always:
  - Limit voice recording to max 60 seconds (frontend + backend)
  - Reject audio files > 5MB
  - Use compressed format (WebM/Opus, not WAV)

### Quality
- All endpoints must have at least one happy-path integration test.
- All complex business logic (scoring, clip generation) must have unit tests.
- Frontend components used in critical flows must have a smoke test.
- Type coverage > 90% on backend (Python type hints), 100% on frontend (TS strict).

---

## 🧪 Testing Strategy

### Backend (Python)
- Framework: `pytest`
- Coverage tool: `pytest-cov`
- Required for: services, scoring logic, API endpoints
- Run: `pytest backend/tests/`

### Frontend (TypeScript)
- Framework: `vitest` for unit tests, `playwright` for E2E
- Required for: scoring display logic, audio recording component, share flow
- Run: `npm test` and `npm run e2e`

### Don't bother testing
- Database migrations (they're tested by running them)
- Pure presentation components (no logic)
- Third-party library wrappers

---

## 🤖 Multi-Agent Coordination

When both Claude Code and Codex are working:

- **Communicate via CHANGELOG.md.** Each merged PR adds an entry.
- **Avoid file collisions.** If working on the same area, coordinate via the owner.
- **Honor each other's code style.** If Claude wrote a service in pattern X, Codex should follow X for similar services.
- **One source of truth: this repo.** Don't introduce alternative patterns mid-stream.

---

## ❓ When to Ask the Owner

Always ask when:
- A spec is ambiguous or contradicts another spec.
- You need a new dependency that's not in the existing stack.
- You're about to introduce a new pattern, framework, or major refactor.
- Cost implications: a feature would push LLM/STT costs above Rp 200K/day.
- Security implications: any change to auth, payment, or data handling.
- You believe a feature should be deferred or descoped.

Never ask for:
- Implementation detail choices that don't affect API or UX.
- Code style decisions covered in `03_conventions.md`.
- Library version bumps within the same major version.

---

## 📝 Commit Style

Follow Conventional Commits:

```
feat(auth): add jwt middleware
fix(scoring): correct emosi dimension calculation
docs(api): update debate endpoint signature
test(scoring): add edge case for empty transcript
refactor(workers): extract whisper client to service
chore(deps): bump fastapi to 0.115.0
```

Scope is the area: `auth`, `debate`, `scoring`, `clip`, `frontend`, `infra`, etc.

Keep commits small and focused. One feature per PR.

---

## 🚨 Critical Reminders

1. **MVP scope is locked.** No party system, no elections, no career until v2 is approved.
2. **Bahasa Indonesia for UI, English for code.** No Indonesian variable names.
3. **Voice files are temporary.** Delete after processing. Privacy matters.
4. **Cost is a feature.** A feature that triples our LLM bill is not worth shipping.
5. **Test the AI scoring.** It's the heart of the product. Bad scoring = dead game.

---

*If you are an AI agent reading this: confirm you have read this file and the linked docs before producing code. Cite specific spec sections in your reasoning.*
