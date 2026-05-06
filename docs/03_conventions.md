# 03 — Coding Conventions

## General Principles

1. **Readability over cleverness.** Junior developers should understand the code in 6 months.
2. **Explicit over implicit.** No magic. Type everything you can.
3. **Functions over classes** unless state is genuinely needed.
4. **Async by default** in backend.
5. **Composition over inheritance** in both frontend and backend.

---

## Backend (Python / FastAPI)

### Project Structure

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app entry point
│   ├── config.py               # Settings via pydantic-settings
│   ├── database.py             # DB connection, session factory
│   ├── deps.py                 # Common dependencies (current_user, db, etc)
│   ├── exceptions.py           # Custom exception classes + handlers
│   │
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── router.py           # FastAPI router (endpoints)
│   │   ├── service.py          # Business logic
│   │   ├── schemas.py          # Pydantic request/response models
│   │   ├── models.py           # SQLAlchemy models
│   │   └── jwt.py              # Token utilities
│   │
│   ├── debates/                # Same structure as auth/
│   ├── topics/
│   ├── personas/
│   ├── users/
│   ├── clips/
│   │
│   └── core/
│       ├── llm.py              # OpenRouter client
│       ├── whisper.py          # Whisper API client
│       ├── storage.py          # MinIO client
│       ├── ratelimit.py        # Rate limiting utilities
│       └── moderation.py       # Content moderation
│
├── workers/
│   ├── __init__.py
│   ├── celery_app.py           # Celery initialization
│   ├── tasks/
│   │   ├── transcribe.py
│   │   ├── score.py
│   │   ├── opponent.py
│   │   ├── clip.py
│   │   └── cleanup.py
│   └── ffmpeg_helpers.py
│
├── migrations/                  # Alembic migrations
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_debates.py
│   └── ...
│
├── pyproject.toml
├── requirements.txt
└── Dockerfile
```

### Naming

- Modules and files: `snake_case.py`
- Classes: `PascalCase`
- Functions and variables: `snake_case`
- Constants: `UPPER_SNAKE_CASE`
- Private (module-internal): leading `_underscore`
- Type aliases: `PascalCase` (e.g., `UserId = int`)

### Type Hints

Mandatory on all public functions and class methods. Optional on private helpers.

```python
from typing import Optional
from uuid import UUID

async def get_debate_by_id(
    db: AsyncSession,
    debate_id: UUID,
    user_id: int,
) -> Optional[Debate]:
    """Fetch a debate, ensuring it belongs to the user."""
    ...
```

### Pydantic Schemas

Separate schemas for request, response, and database models.

```python
# schemas.py
class DebateCreate(BaseModel):
    topic_id: UUID
    persona_id: UUID
    position: Literal["PRO", "KONTRA"]

class DebateResponse(BaseModel):
    id: UUID
    topic: TopicResponse
    persona: PersonaResponse
    position: str
    status: str
    scores: Optional[ScoreBreakdown] = None
    transcript: Optional[str] = None
    opponent_response: Optional[str] = None
    clip_url: Optional[str] = None
    created_at: datetime
    
    model_config = ConfigDict(from_attributes=True)
```

### Service Pattern

Routes are thin. Business logic lives in services.

```python
# router.py
@router.post("/debates", response_model=DebateResponse)
async def create_debate(
    data: DebateCreate,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> DebateResponse:
    debate = await debate_service.create_debate(db, user.id, data)
    return debate

# service.py
async def create_debate(
    db: AsyncSession,
    user_id: int,
    data: DebateCreate,
) -> Debate:
    """Create a new debate session and return with presigned upload URL."""
    # validation, business logic, DB operations
    ...
```

### Error Handling

Use custom exceptions, handled by global exception handlers.

```python
# exceptions.py
class AppException(Exception):
    status_code: int = 400
    message: str = "Something went wrong"

class NotFoundError(AppException):
    status_code = 404
    message = "Resource not found"

class RateLimitError(AppException):
    status_code = 429
    message = "Too many requests"

# main.py
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.message},
    )
```

Never raise generic `Exception`. Never return `{"error": "..."}` directly from a route — use exceptions.

### Database Access

Use SQLAlchemy 2.0 async style.

```python
async def get_user_by_email(db: AsyncSession, email: str) -> Optional[User]:
    stmt = select(User).where(User.email == email)
    result = await db.execute(stmt)
    return result.scalar_one_or_none()
```

No raw SQL except for migrations or extreme performance cases. Always use ORM.

### Testing

```python
# tests/test_debates.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_debate_returns_upload_url(
    client: AsyncClient,
    authenticated_user: User,
    seed_topics,
    seed_personas,
):
    response = await client.post(
        "/api/debates",
        json={
            "topic_id": str(seed_topics[0].id),
            "persona_id": str(seed_personas[0].id),
            "position": "PRO",
        },
    )
    assert response.status_code == 201
    body = response.json()
    assert "upload_url" in body
    assert body["status"] == "pending"
```

### Linting

- `ruff` for lint + format (replaces black, flake8, isort)
- `mypy --strict` for type checking
- Run before every commit: `ruff check . && ruff format . && mypy app/`

### Logging

Use Python's standard `logging`. Configure via dictConfig in `main.py`.

```python
import logging
logger = logging.getLogger(__name__)

async def transcribe_audio(debate_id: UUID):
    logger.info("Starting transcription", extra={"debate_id": str(debate_id)})
    try:
        # ...
        logger.info("Transcription complete", extra={"debate_id": str(debate_id), "duration_ms": ms})
    except WhisperAPIError as e:
        logger.error("Transcription failed", extra={"debate_id": str(debate_id), "error": str(e)})
        raise
```

Log levels:
- `DEBUG` — Detailed diagnostic info, never in prod
- `INFO` — Normal events (request start/end, task complete)
- `WARNING` — Unexpected but handled (rate limit hit, fallback used)
- `ERROR` — Failures requiring attention
- `CRITICAL` — Service-down events

---

## Frontend (TypeScript / React)

### Project Structure

```
frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── routes.tsx               # React Router config
│   │
│   ├── components/
│   │   ├── ui/                  # shadcn/ui components (auto-generated)
│   │   ├── auth/
│   │   │   ├── LoginForm.tsx
│   │   │   └── RegisterForm.tsx
│   │   ├── debate/
│   │   │   ├── TopicSelector.tsx
│   │   │   ├── PersonaSelector.tsx
│   │   │   ├── AudioRecorder.tsx
│   │   │   ├── ScoreDisplay.tsx
│   │   │   └── ClipPreview.tsx
│   │   └── common/
│   │       ├── Layout.tsx
│   │       ├── Header.tsx
│   │       └── ErrorBoundary.tsx
│   │
│   ├── pages/
│   │   ├── HomePage.tsx
│   │   ├── LoginPage.tsx
│   │   ├── DebatePage.tsx
│   │   ├── ResultPage.tsx
│   │   └── HistoryPage.tsx
│   │
│   ├── lib/
│   │   ├── api.ts               # API client (fetch wrapper)
│   │   ├── auth.ts              # Auth helpers
│   │   ├── audio.ts             # MediaRecorder wrapper
│   │   └── utils.ts
│   │
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useDebate.ts
│   │   └── useAudioRecorder.ts
│   │
│   ├── stores/
│   │   ├── authStore.ts         # Zustand: auth state
│   │   └── debateStore.ts       # Zustand: current debate state
│   │
│   ├── types/
│   │   └── api.ts               # TypeScript types matching backend schemas
│   │
│   └── styles/
│       └── globals.css
│
├── public/
├── package.json
├── tsconfig.json
├── vite.config.ts
└── tailwind.config.js
```

### TypeScript Strictness

`tsconfig.json` must have:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true
  }
}
```

No `any`. If absolutely needed, use `unknown` and narrow.

### Component Patterns

Functional components only. Hooks for logic.

```typescript
// AudioRecorder.tsx
interface AudioRecorderProps {
  maxDurationSec: number;
  onRecordingComplete: (audioBlob: Blob) => void;
  onCancel: () => void;
}

export function AudioRecorder({
  maxDurationSec,
  onRecordingComplete,
  onCancel,
}: AudioRecorderProps) {
  const { isRecording, duration, start, stop, cancel } = useAudioRecorder({
    maxDurationSec,
    onComplete: onRecordingComplete,
  });

  return (
    <div className="flex flex-col items-center gap-4">
      <Timer duration={duration} max={maxDurationSec} />
      {isRecording ? (
        <Button onClick={stop} variant="destructive">Stop</Button>
      ) : (
        <Button onClick={start} variant="default">Start Recording</Button>
      )}
    </div>
  );
}
```

### State Management

- **Server state** (data from API): TanStack Query
- **Client state** (UI, ephemeral): React `useState`
- **Global client state** (auth, current session): Zustand

Never put server state in Zustand. Never put UI state in TanStack Query.

```typescript
// Good: server state via TanStack Query
function HistoryPage() {
  const { data: debates, isLoading } = useQuery({
    queryKey: ['debates', 'history'],
    queryFn: api.debates.getHistory,
  });
  // ...
}

// Good: ephemeral UI state via useState
function DebatePage() {
  const [step, setStep] = useState<'select' | 'record' | 'processing' | 'result'>('select');
  // ...
}

// Good: global auth state via Zustand
const useAuth = create<AuthStore>(...);
```

### API Client

Single `lib/api.ts` file. Typed functions matching backend schemas.

```typescript
// lib/api.ts
import { Debate, DebateCreate, ScoreBreakdown } from '@/types/api';

const API_URL = import.meta.env.VITE_API_URL ?? '/api';

class ApiClient {
  async createDebate(data: DebateCreate): Promise<Debate> {
    const response = await fetch(`${API_URL}/debates`, {
      method: 'POST',
      credentials: 'include',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!response.ok) throw new ApiError(response);
    return response.json();
  }
  // ...
}

export const api = new ApiClient();
```

### Styling

- TailwindCSS for all styling.
- Use `clsx` for conditional classes.
- Use shadcn/ui for primitive components (Button, Dialog, Card, etc).
- Custom design tokens in `tailwind.config.js`.
- No CSS files except `globals.css` for resets.

### Indonesian UI Text

All user-facing text in Bahasa Indonesia. Centralize in a constants file for now (full i18n is v2).

```typescript
// lib/text.ts
export const TEXT = {
  buttons: {
    startDebate: 'Mulai Debat',
    record: 'Mulai Rekam',
    stop: 'Stop',
    submit: 'Kirim',
    cancel: 'Batal',
    share: 'Bagikan',
    download: 'Unduh',
  },
  status: {
    transcribing: 'Mentranskrip suara...',
    scoring: 'AI sedang menilai...',
    generatingClip: 'Membuat klip highlight...',
    complete: 'Selesai!',
  },
  errors: {
    micPermissionDenied: 'Izin akses mikrofon ditolak. Silakan beri izin di pengaturan browser.',
    audioTooShort: 'Rekaman terlalu pendek. Minimal 10 detik.',
    audioTooLong: 'Rekaman terlalu panjang. Maksimal 60 detik.',
    networkError: 'Koneksi bermasalah. Coba lagi.',
  },
};
```

### Linting

- `eslint` with `@typescript-eslint`
- `prettier` for formatting
- Run before commit: `npm run lint && npm run format && npm run typecheck`

### Testing

- `vitest` for unit tests
- `@testing-library/react` for component tests
- `playwright` for E2E (only critical flows)

```typescript
// AudioRecorder.test.tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { AudioRecorder } from './AudioRecorder';

describe('AudioRecorder', () => {
  it('shows start button initially', () => {
    render(<AudioRecorder maxDurationSec={60} onRecordingComplete={vi.fn()} onCancel={vi.fn()} />);
    expect(screen.getByText('Start Recording')).toBeInTheDocument();
  });
});
```

---

## Git Workflow

### Branches

- `main` — Always deployable.
- `dev` — Integration branch.
- `feat/<short-description>` — Feature branches off `dev`.
- `fix/<short-description>` — Bug fix branches.
- `chore/<short-description>` — Tooling, deps.

### Commits

Conventional Commits format. Already specified in `AGENTS.md`.

### PRs

- One feature per PR.
- PR description must reference the spec it implements (e.g., "Implements spec/02_debate_session.md").
- All checks must pass (lint, types, tests).
- Squash merge into `dev`.

### Deployment

Deploy from `main`. Manual for now (SSH + git pull + docker compose up). CI/CD in week 8.

---

## Documentation

- README at repo root explains setup and high-level usage.
- Each module has a docstring at the top explaining its purpose.
- Complex functions have inline comments explaining *why*, not *what*.
- API endpoints documented via FastAPI's auto-generated OpenAPI (Swagger UI at `/docs`).
- Frontend components documented via JSDoc on the component function (used by IDEs).

Avoid:
- Comments that restate the code.
- Redundant docstrings (`# This function returns x` for `def returns_x():`).
- TODO comments without owner and ticket reference.

---

## Performance Budgets

### Backend
- p50 response time: < 100ms (excluding LLM/STT calls)
- p95 response time: < 500ms (excluding LLM/STT calls)
- LLM scoring: < 8s end-to-end
- STT transcription: < 5s for 60s audio

### Frontend
- First Contentful Paint: < 1.5s
- Time to Interactive: < 3s
- JS bundle (gzipped): < 400KB
- Audio recording start latency: < 200ms after click

If a feature blows the budget, flag it. Don't ship slow code silently.
