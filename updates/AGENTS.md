# updates/ — Incremental Update Specs

> **For AI agents (Claude Code / Codex):** Files in this folder are specs for ADDITIVE updates to a running production app. Different rules apply than `specs/` folder.

## How to Read These

Each `updates/NN_*.md` file describes a feature being added to the existing application. The application is already running on VPS with users — these are not greenfield specs.

Before implementing:

1. Read the update file fully
2. Cross-reference with existing code to understand current state
3. Verify that "Out of Scope" items are not silently included
4. Check if any existing files would be modified — if yes, FLAG and ASK before proceeding

## Critical Rules for Update Specs

### Additive Only
- New tables, new columns: OK
- New endpoints, new modules: OK
- Modifying existing table columns: REQUIRES OWNER APPROVAL
- Modifying existing endpoints' response shape: BREAKING CHANGE — flag explicitly
- Removing existing functionality: NEVER without explicit instruction

### Migration Safety
- All DB migrations must be backward-compatible (old code can run against new schema)
- Test rollback path for every migration
- Run migrations in transaction when possible
- Backfill existing data explicitly (don't leave NULL where new code expects values)

### Feature Flags
- Wrap new behavior in feature flags from `.env`
- Default to OFF in `.env.example`
- Flag name pattern: `FEATURE_<UPDATE_NAME>_ENABLED`
- Allow gradual rollout: per-user, percentage-based, or all-or-nothing

### Don't Break Existing Tests
- All existing tests must pass after the update
- If existing test must change, flag and explain why
- Add new tests for new behavior
- Don't reduce test coverage

## Numbering

Updates are numbered sequentially: `01_*`, `02_*`, `03_*`. Implement in order. If owner approves out-of-order implementation, document the dependency in the update file.

## Order of Operations Within Each Update

Each update typically follows this sequence:

1. Database migration (with backfill)
2. Model definitions (SQLAlchemy)
3. Service layer (business logic)
4. API endpoints
5. Worker tasks (if any)
6. Frontend types
7. Frontend components
8. Frontend integration with existing pages
9. Tests at each layer

Don't ship partial — each PR should include backend + tests at minimum. Frontend can be separate PR if backend is feature-flagged.

## Communication

- Update `CHANGELOG.md` after each merged update PR
- Reference the update file in the PR description: "Implements `updates/01_engagement_layer.md`"
- If you discover a flaw in the update spec while implementing, flag it in the PR — don't silently deviate

---

## Current Update Index

| File | Status | Description |
|---|---|---|
| `01_engagement_layer.md` | Pending | League tiers + LP, streaks, daily missions, adaptive difficulty, soft limits |

(Add new entries here as updates are added.)
