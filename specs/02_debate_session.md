# Spec 02 — Debate Session (Core Gameplay)

## Purpose

The core gameplay flow: user selects a topic and persona, records voice argument, gets AI scoring with feedback, and gets a shareable highlight clip.

This is the most important feature in the MVP. Get it right.

## User Story

> Sebagai pemain yang sudah login, saya ingin memilih sebuah topik politik, memilih lawan AI, kemudian merekam argumen saya selama maksimum 60 detik. Setelah itu, saya ingin melihat skor saya, mendapat feedback yang berguna, melihat tanggapan dari lawan AI, dan mendownload klip highlight untuk dibagikan ke media sosial.

## Flow Steps

### Step 1: Topic Selection
User clicks "Mulai Debat" on home page → navigates to topic selection.

UI shows:
- Filter dropdowns: kategori, kesulitan
- Grid of topic cards (5-15 visible)
- Each card: motion text, kategori badge, kesulitan badge
- Click card to select

After click, navigate to position selection.

### Step 2: Position Selection
User sees the selected topic prominently. Two big buttons:
- "PRO — Saya Setuju"
- "KONTRA — Saya Tidak Setuju"

Click one, navigate to persona selection.

### Step 3: Persona Selection
Three persona cards displayed:
- Si Profesor (Sulit, target 80, +45 ELO)
- Si Pak RT (Sedang, target 70, +30 ELO)
- Si Aktivis 98 (Sulit, target 80, +50 ELO)

Each card: name, archetype, tagline, difficulty, target score, reward preview.

Click one, navigate to briefing screen.

### Step 4: Briefing
Shows summary:
- Topik: [motion]
- Posisi: [PRO/KONTRA]
- Lawan: [persona name]

Briefing text (static, in Bahasa Indonesia):
- "Anda akan merekam argumen selama maksimum 60 detik"
- "Bicarakan dengan jelas dan terstruktur"
- "AI akan menilai 5 dimensi: Logika, Data, Emosi, Konsistensi, Retorika"

Mic check section:
- Click "Tes Mikrofon" → records 3 seconds, plays back
- Visual confirmation mic works

Bottom CTA: "Mulai Rekam" (disabled until mic check done)

Click → backend creates debate, returns presigned URL → navigate to recording screen.

### Step 5: Recording
UI:
- Topik display at top (compact)
- Persona display (small avatar + name)
- Large central element: countdown timer (60 → 0)
- Live waveform visualization (animated bars)
- Big record/stop button
- "Batalkan" link at bottom

Behavior:
- Click "Mulai Rekam" → starts MediaRecorder, timer starts
- During recording: timer counts down, waveform animates
- User can click "Stop" to end early
- Auto-stops at 0 seconds
- After stop: brief "Mengirim..." state, audio uploads to MinIO
- After upload: backend notified, navigate to processing screen

### Step 6: Processing
UI shows animated loading screen:
- "AI sedang menganalisis argumen Anda..."
- Progress indicator (visual, doesn't need to be accurate)
- Shows current step:
  - "Mentranskripsi suara..." (transcribing)
  - "Menilai argumen..." (scoring)
  - "Membuat tanggapan AI..." (generating_response)
  - "Membuat klip highlight..." (generating_clip)
- Frontend polls `GET /api/debates/{id}/status` every 2 seconds

If error occurs (status='failed'):
- Show friendly error message based on error_message field
- "Coba Lagi" button → resubmits with new debate session

If success (status='complete'):
- Auto-navigate to result screen

### Step 7: Result
The reveal screen. Important to make this feel rewarding.

Layout (top to bottom):

**Header banner:**
- "MENANG!" / "KALAH" / "SERI" badge with appropriate color (green/red/yellow)
- Skor total in large font (animated count-up from 0)
- "Skor 78/100" or similar

**Score breakdown:**
- 5 dimension cards in a row (or grid on mobile)
- Each card: dimension name, score (0-10), mini progress bar
- Color-coded: green ≥7.5, yellow 6.0-7.4, red <6.0

**AI Feedback section:**
- "Catatan AI Juri" header
- 5 feedback texts (one per dimension), expandable accordion
- Overall feedback at bottom (always visible)

**Opponent Response section:**
- "Tanggapan Lawan: [persona name]"
- Quote-style display of AI's counter-argument
- Persona avatar/icon

**Clip section:**
- "Klip Highlight Anda" header
- Video player (16:9 or 9:16 vertical)
- Play button overlay
- Duration: 18s typical
- Below: share buttons (TikTok, Instagram, WhatsApp), Download button, Copy link

**Bottom actions:**
- "Debat Lagi" (primary CTA, returns to topic selection)
- "Lihat Riwayat" (secondary)

### Step 8: Share/Download
Click share button → opens native share sheet (mobile) or copies pre-formatted text (desktop).

Click download → triggers MP4 download via signed URL.

## Backend Implementation

### Module Structure
```
backend/app/debates/
├── __init__.py
├── router.py
├── service.py
├── schemas.py
├── models.py
└── exceptions.py

backend/workers/tasks/
├── transcribe.py
├── score.py
├── opponent.py
├── clip.py
└── cleanup.py
```

### Endpoints
See `docs/05_api_design.md`:
- `POST /api/debates` — Create
- `POST /api/debates/{id}/submit` — Notify upload complete
- `GET /api/debates/{id}/status` — Poll status
- `GET /api/debates/{id}` — Get full result
- `GET /api/debates` — History list

### Status State Machine

```
                  pending
                     │ (user uploads + submit endpoint hit)
                     ▼
                transcribing
                     │
            ┌────────┴───────────┐
            │ success            │ failed
            ▼                    ▼
         scoring              failed
            │
   ┌────────┴───────────┐
   │ success            │ failed
   ▼                    ▼
generating_response   failed
   │
   ┌────────┴───────────┐
   │ success            │ failed
   ▼                    ▼
generating_clip       failed
   │
   ┌────────┴───────────┐
   │ success            │ failed
   ▼                    ▼
complete              failed
```

Each transition is an explicit DB update by the worker.

### Celery Task Chain

When `/submit` is called, enqueue:
```python
chain(
    transcribe.s(debate_id),
    score.s(),  # receives debate_id from previous
    generate_opponent.s(),
    generate_clip.s(),
    cleanup_audio.s(),
).apply_async()
```

If any task in chain fails, the chain stops and `score_debate_failed` callback updates DB to `failed` status.

### Idempotency
All tasks must be idempotent (safe to retry):
- Transcribe: check if transcript already exists; if yes, skip
- Score: check if score already exists; if yes, skip
- Opponent: check if response already exists; if yes, skip
- Clip: check if clip URL already exists; if yes, skip
- Cleanup: check if audio_url is already null; if yes, skip

This allows manual retry of failed debates without double-charging.

## Frontend Implementation

### Pages
```
frontend/src/pages/
├── DebatePage.tsx          # Multi-step flow controller
└── ResultPage.tsx          # Post-debate result view

frontend/src/components/debate/
├── TopicSelector.tsx
├── PositionSelector.tsx
├── PersonaSelector.tsx
├── BriefingScreen.tsx
├── AudioRecorder.tsx       # Already in week 4
├── ProcessingScreen.tsx
├── ScoreDisplay.tsx
├── FeedbackAccordion.tsx
├── OpponentResponse.tsx
├── ClipPreview.tsx
└── ShareButtons.tsx
```

### State Management
Use Zustand for the active debate flow:
```typescript
interface DebateFlowState {
  step: 'topic' | 'position' | 'persona' | 'briefing' | 'recording' | 'processing' | 'result';
  selectedTopic: Topic | null;
  selectedPosition: 'PRO' | 'KONTRA' | null;
  selectedPersona: Persona | null;
  debateId: string | null;
  goToStep: (step: DebateStep) => void;
  reset: () => void;
}
```

After result, reset state when user clicks "Debat Lagi".

### Polling Logic
```typescript
function useDebateStatus(debateId: string) {
  return useQuery({
    queryKey: ['debate', debateId, 'status'],
    queryFn: () => api.debates.getStatus(debateId),
    refetchInterval: (data) => {
      if (data?.status === 'complete' || data?.status === 'failed') {
        return false;  // Stop polling
      }
      return 2000;  // Poll every 2s
    },
  });
}
```

When status becomes 'complete', fetch full debate details and navigate to result page.

### Audio Recording (Already Built in Week 4)
Reuse `AudioRecorder` component. Add prop `onUploaded` that fires after successful MinIO upload.

```typescript
async function handleRecorded(blob: Blob) {
  const debate = await api.debates.create(...);
  await uploadToMinio(debate.upload_url, blob);
  await api.debates.submit(debate.id);
  goToStep('processing');
}
```

## Edge Cases

### Audio Recording Edge Cases
1. **Mic permission denied**: Show error, link to browser settings.
2. **No microphone detected**: Show error, prompt to plug one in.
3. **Recording too short** (<10s): Reject, show "Argumen terlalu pendek. Coba lagi."
4. **Recording empty** (silence detected by waveform): Reject, show "Tidak ada suara terdeteksi."
5. **Browser doesn't support MediaRecorder**: Show fallback message (very rare in 2026).

### Network Edge Cases
1. **Upload fails midway**: Retry once. If still fails, mark debate failed, allow retry.
2. **Status polling fails**: Retry with exponential backoff, max 3 retries, then show "Koneksi bermasalah".
3. **User closes tab during processing**: Backend continues processing. User can return to history page later and see result.

### LLM Edge Cases
1. **All LLM providers down**: Status becomes failed with friendly message. User can retry later (no charge).
2. **LLM returns invalid JSON**: Retry once with stricter prompt. If still fails, mark failed.
3. **Score outside expected range**: Clamp to 0-10. Log warning.
4. **Transcript flagged as offensive**: Block scoring, show warning to user.

### Database Edge Cases
1. **Daily limit reached**: Show "Anda sudah mencapai batas harian. Datang lagi besok atau upgrade ke premium."
2. **Topic deactivated mid-debate**: Allow current debate to complete (soft check, hard check at create only).

## Acceptance Criteria

- [ ] User can navigate from home to result in <5 minutes (with 60s recording)
- [ ] All steps work without errors on Chrome, Firefox, Safari
- [ ] Recording works on mobile browsers (Android Chrome, iOS Safari)
- [ ] Status polling shows correct progress per stage
- [ ] Result page renders all sections (scores, feedback, opponent, clip)
- [ ] Clip is playable in browser without external player
- [ ] Share buttons work and pre-fill text correctly
- [ ] Failed debates show clear error and allow retry
- [ ] Daily limit enforced and message shown
- [ ] All UI text in Bahasa Indonesia
- [ ] Performance: full flow loads in <3s on 4G connection
- [ ] Tests pass (backend + frontend)

## Things to Defer to v2

- Saving draft debates (closing tab loses progress)
- Multi-round debates (just 1 round for MVP)
- Voice-based AI opponent response (text only for MVP)
- Real-time emotion analysis during recording
- Pause/resume during recording
- Multiple takes (one shot only for MVP)
