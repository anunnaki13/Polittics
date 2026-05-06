# 07 — AI Pipeline (STT + LLM Scoring)

## Overview

The AI pipeline is the heart of the product. It turns a 60-second voice recording into a multi-dimensional score with feedback. This document explains how.

## Pipeline Stages

```
Voice Recording (WebM)
         │
         ▼
    [Whisper STT]   ───>   Transcript (Indonesian text)
         │
         ▼
    [Validation]    ───>   Reject if too short, gibberish, or moderation flag
         │
         ▼
   [LLM Scoring]    ───>   5 dimensions + feedback (JSON)
         │
         ▼
[Opponent Generation] ─>   AI counter-argument text
         │
         ▼
   [Clip Generation]  ─>   15-25s MP4 with audio + visuals
         │
         ▼
       Result
```

Each stage is a separate Celery task, allowing retry/recovery if any fails.

---

## Stage 1: Whisper STT

### Configuration

```python
# app/core/whisper.py
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)

async def transcribe_audio(
    audio_bytes: bytes,
    filename: str = "audio.webm",
) -> WhisperResult:
    response = await client.audio.transcriptions.create(
        model="whisper-1",
        file=(filename, audio_bytes, "audio/webm"),
        language="id",  # Indonesian
        response_format="verbose_json",  # Includes confidence scores
        temperature=0.0,  # Deterministic
    )
    return WhisperResult(
        text=response.text,
        confidence=calculate_avg_confidence(response.segments),
        duration=response.duration,
        segments=response.segments,
    )
```

### Cost

- $0.006 per minute of audio
- Per session (60s max): $0.006
- Per 1000 sessions: $6.00 = Rp 96K

### Failure Handling

If Whisper API fails:
1. Retry once after 5s
2. If still fails, fall back to local Whisper-tiny model (slower, less accurate)
3. If both fail, mark debate `failed` with error_message

Local Whisper fallback:
```python
import whisper
model = whisper.load_model("tiny")  # ~75MB

async def transcribe_local(audio_bytes: bytes) -> WhisperResult:
    # Save to temp file, run local model
    ...
```

### Validation After Transcription

```python
def validate_transcript(transcript: str) -> ValidationResult:
    # Reject if too short
    if len(transcript.strip()) < 30:
        return ValidationResult(valid=False, reason="too_short")
    
    # Reject if too short for duration (someone recording silence)
    word_count = len(transcript.split())
    if word_count < 20:
        return ValidationResult(valid=False, reason="too_few_words")
    
    # Reject if mostly non-Indonesian (heuristic)
    if not is_likely_indonesian(transcript):
        return ValidationResult(valid=False, reason="not_indonesian")
    
    return ValidationResult(valid=True)
```

If invalid, debate is marked `failed` and user sees a friendly message ("Rekaman tidak terdeteksi sebagai argumen Bahasa Indonesia. Coba lagi.").

---

## Stage 2: Content Moderation

Before scoring, run a quick moderation check on the transcript.

```python
async def moderate_content(transcript: str) -> ModerationResult:
    # Use cheap fast model for moderation
    response = await llm_call(
        model="google/gemini-2.5-flash",
        system="You are a content moderator for an Indonesian political debate game.",
        user=f"""Transkripsi: {transcript}

Apakah teks ini mengandung salah satu hal berikut?
1. Ujaran kebencian SARA (suku, agama, ras, antar-golongan)
2. Penyebutan nama tokoh politik nyata Indonesia
3. Penyebutan nama partai politik nyata Indonesia
4. Pencemaran nama baik
5. Konten tidak senonoh

Jawab dalam JSON: {{"flagged": boolean, "reasons": [list of reasons]}}""",
        max_tokens=200,
        temperature=0.0,
    )
    return parse_moderation_response(response)
```

If flagged, debate proceeds but score is reduced and warning shown to user. Severe violations (hate speech) result in failed status.

---

## Stage 3: LLM Scoring

This is the most critical and complex part of the system. Get this right.

### Scoring Prompt Structure

The full prompt is in `prompts/scoring_prompt.md`. Summary:

```
SYSTEM PROMPT:
You are an expert AI judge for political debates in Indonesia. Your task is 
to score a player's argument on 5 dimensions, providing fair and constructive 
feedback. Output strictly in JSON format.

USER PROMPT:
TOPIK: {topic.motion}
POSISI PEMAIN: {position}  // PRO or KONTRA
PERSONA LAWAN: {persona.name} (target skor: {persona.target_score})

TRANSKRIPSI ARGUMEN PEMAIN:
{transcript}

DURASI: {duration_sec} detik

Nilai argumen pemain pada 5 dimensi (skor 0-10 per dimensi):
1. LOGIKA - Struktur argumen, validitas premis-konklusi, deteksi fallacy
2. DATA - Akurasi fakta, kutipan sumber, relevansi statistik
3. EMOSI - Kontrol nada, variasi intonasi (inferred dari pemilihan kata dan struktur)
4. KONSISTENSI - Tidak kontradiksi internal, stick to position
5. RETORIKA - Pemilihan diksi, struktur kalimat, analogi, kalimat memorable

OUTPUT FORMAT (JSON):
{
  "logika": {"score": 0-10, "feedback": "1-2 kalimat"},
  "data": {"score": 0-10, "feedback": "1-2 kalimat"},
  "emosi": {"score": 0-10, "feedback": "1-2 kalimat"},
  "konsistensi": {"score": 0-10, "feedback": "1-2 kalimat"},
  "retorika": {"score": 0-10, "feedback": "1-2 kalimat"},
  "feedback_overall": "2-4 kalimat ringkasan dan saran perbaikan"
}
```

### Implementation

```python
# workers/tasks/score.py
async def score_debate(debate_id: UUID):
    debate = await get_debate(debate_id)
    
    prompt = build_scoring_prompt(
        topic=debate.topic,
        persona=debate.persona,
        position=debate.position,
        transcript=debate.transcript,
        duration_sec=debate.audio_duration_sec,
    )
    
    raw_response = await llm_call(
        model=settings.LLM_PRIMARY_MODEL,
        system=SCORING_SYSTEM_PROMPT,
        user=prompt,
        max_tokens=1500,
        temperature=0.3,
        response_format={"type": "json_object"},
    )
    
    parsed = parse_scoring_response(raw_response)
    
    # Calculate weighted total
    total = (
        parsed.logika * 2.5 +
        parsed.data * 2.0 +
        parsed.emosi * 1.5 +
        parsed.konsistensi * 2.0 +
        parsed.retorika * 2.0
    )  # Total max = 100
    
    # Determine result
    target = debate.persona.target_score
    if total > target + 2:
        result = "MENANG"
    elif total < target - 2:
        result = "KALAH"
    else:
        result = "SERI"
    
    # Save to DB
    await save_scores(debate_id, parsed, total, result)
    
    # Calculate XP
    xp_earned = calculate_xp(total, result, debate.persona.difficulty)
    await update_user_xp(debate.user_id, xp_earned)
```

### Cost Per Scoring Call

Average inputs:
- Scoring prompt template: ~500 tokens
- Transcript (60s ~150 words): ~200 tokens
- Topic + persona context: ~300 tokens
- Total input: ~1000 tokens

Average output:
- JSON response: ~600 tokens

Per call with Gemini 2.5 Flash:
- Input: 1000 tokens × $0.15/1M = $0.00015
- Output: 600 tokens × $0.60/1M = $0.00036
- Total: ~$0.0005

Per 1000 sessions: $0.50 = Rp 8K. Very cheap.

If using Claude Haiku 4.5 (premium):
- Input: 1000 × $1.00/1M = $0.001
- Output: 600 × $5.00/1M = $0.003
- Total: $0.004
- Per 1000: $4 = Rp 64K. Still affordable.

### Multi-Model Strategy

```python
async def llm_call_with_fallback(messages, **kwargs):
    models = [
        settings.LLM_PRIMARY_MODEL,    # Gemini 2.5 Flash
        settings.LLM_FALLBACK_MODEL,   # Claude Haiku 4.5
        "deepseek/deepseek-chat",       # Free tier emergency
    ]
    
    for i, model in enumerate(models):
        try:
            return await openrouter_call(model=model, messages=messages, **kwargs)
        except (RateLimitError, ServiceUnavailableError) as e:
            logger.warning(f"Model {model} failed: {e}, trying fallback")
            if i == len(models) - 1:
                raise
            continue
```

### Validation of LLM Response

LLMs sometimes return invalid JSON or scores outside 0-10. Validate strictly:

```python
def parse_scoring_response(raw: str) -> ScoringResult:
    try:
        data = json.loads(raw)
    except json.JSONDecodeError:
        # Try to extract JSON from markdown code block
        data = extract_json_from_text(raw)
    
    # Validate structure
    required_keys = ["logika", "data", "emosi", "konsistensi", "retorika", "feedback_overall"]
    for key in required_keys:
        if key not in data:
            raise InvalidLLMResponseError(f"Missing key: {key}")
    
    # Validate score ranges
    for dim in ["logika", "data", "emosi", "konsistensi", "retorika"]:
        score = data[dim].get("score")
        if not isinstance(score, (int, float)):
            raise InvalidLLMResponseError(f"Invalid score type for {dim}")
        if not 0 <= score <= 10:
            # Clamp instead of error (LLMs occasionally go slightly out of range)
            data[dim]["score"] = max(0, min(10, score))
    
    return ScoringResult(**data)
```

If LLM fails validation 3 times in a row, mark debate as `failed` with error.

---

## Stage 4: Opponent Response Generation

After scoring, generate the AI persona's counter-argument. This is shown to user as "Tanggapan Lawan".

### Prompt Structure

See `prompts/opponent_prompts.md` for full persona-specific prompts. Skeleton:

```
SYSTEM PROMPT (persona-specific):
You are {persona.name}, a {persona.archetype} debater. {persona.system_prompt}

USER PROMPT:
TOPIK: {topic.motion}
POSISI ANDA: {opposite_position}  // If user PRO, AI is KONTRA
ARGUMEN PEMAIN: {transcript}

Berikan tanggapan singkat 80-120 kata yang:
1. Mengakui satu poin valid dari pemain (build trust)
2. Menyerang 2 kelemahan utama dari argumen pemain
3. Mengakhiri dengan pertanyaan/tantangan yang menggugah

Karakter Anda: {persona.tagline}
```

### Implementation

```python
async def generate_opponent_response(debate_id: UUID):
    debate = await get_debate(debate_id)
    
    opposite_position = "KONTRA" if debate.position == "PRO" else "PRO"
    
    response = await llm_call(
        model=settings.LLM_PRIMARY_MODEL,
        system=debate.persona.system_prompt,
        user=build_opponent_prompt(
            persona=debate.persona,
            topic=debate.topic,
            position=opposite_position,
            transcript=debate.transcript,
        ),
        max_tokens=300,
        temperature=0.7,  # More creative than scoring
    )
    
    # Validate length (80-120 words)
    word_count = len(response.split())
    if not 50 <= word_count <= 200:
        logger.warning(f"Opponent response unusual length: {word_count} words")
    
    await save_opponent_response(debate_id, response)
```

### Cost

- Input: ~600 tokens (persona prompt + topic + transcript)
- Output: ~150 tokens (80-120 word response)
- Cost with Gemini Flash: ~$0.0002 per call
- Per 1000: $0.20 = Rp 3K

---

## Stage 5: Clip Generation

Generate a 15-25 second highlight video that combines the user's audio with visual overlays for sharing.

### Approach

Use FFmpeg with pre-made template assets:
- Background: Static gradient image (1080x1920 for vertical/TikTok format)
- Player avatar: User's initials in styled circle
- Topic text: Scrolling banner at top
- Score reveal: Animated number counter
- Waveform: Generated from audio
- Branding: Arena Politika watermark

### Implementation

```python
# workers/tasks/clip.py
import ffmpeg
from pathlib import Path

async def generate_clip(debate_id: UUID):
    debate = await get_debate(debate_id)
    
    # Download audio from MinIO
    audio_path = await download_audio(debate.audio_url)
    
    # Find best 15-25 second segment from transcript
    best_segment = find_best_segment(debate.transcript, debate.audio_duration_sec)
    
    # Build clip
    output_path = f"/tmp/clip_{debate_id}.mp4"
    
    (
        ffmpeg
        .input(BACKGROUND_TEMPLATE_PATH, loop=1, t=best_segment.duration)
        .filter('drawtext', 
                text=debate.topic.motion,
                fontsize=40, fontcolor='white',
                x='(w-text_w)/2', y=100,
                fontfile=FONT_PATH)
        .filter('drawtext',
                text=f"Skor: {debate.scores.total:.0f}",
                fontsize=80, fontcolor='cyan',
                x='(w-text_w)/2', y='h-300',
                enable=f'gte(t,{best_segment.duration - 3})')
        .input(audio_path, ss=best_segment.start, t=best_segment.duration)
        .output(output_path, 
                vcodec='libx264', acodec='aac',
                preset='fast', crf=23,
                **{'b:a': '128k'})
        .overwrite_output()
        .run(quiet=True)
    )
    
    # Upload to MinIO
    clip_url = await upload_clip(output_path, debate_id)
    await save_clip_url(debate_id, clip_url)
    
    # Cleanup
    Path(output_path).unlink()
    Path(audio_path).unlink()
```

### Best Segment Detection (Simplified for MVP)

For MVP, just take the last 18 seconds (closing argument tends to be the most quotable):

```python
def find_best_segment(transcript: str, duration: float) -> Segment:
    if duration <= 25:
        return Segment(start=0, duration=duration)
    
    # Take last 18 seconds
    return Segment(start=duration - 18, duration=18)
```

In v2, use LLM to identify the most impactful sentence and clip around it.

### Output Format

- Resolution: 1080x1920 (vertical, TikTok/Instagram Reels native)
- Codec: H.264 video, AAC audio
- Duration: 15-25 seconds
- File size: ~1.5-3 MB
- Watermark: Bottom-right corner, semi-transparent

---

## Stage 6: Cleanup

After all processing succeeds:

```python
async def cleanup_audio(debate_id: UUID):
    debate = await get_debate(debate_id)
    if debate.status != 'complete':
        return  # Don't delete if still processing
    
    # Delete raw audio from MinIO (transcript is in DB, no longer need audio)
    await delete_object(MINIO_AUDIO_BUCKET, debate.audio_filename)
    
    # Clear audio_url in DB
    await clear_audio_url(debate_id)
```

This is a privacy guarantee. Voice files do not persist.

---

## Total Pipeline Cost Per Session

| Stage | Cost (USD) | Cost (IDR) |
|---|---|---|
| Whisper STT (60s) | $0.006 | Rp 96 |
| Content Moderation | $0.0001 | Rp 1.6 |
| LLM Scoring | $0.0005 | Rp 8 |
| Opponent Response | $0.0002 | Rp 3 |
| Clip Generation | $0 (CPU only) | Rp 0 |
| **Total** | **$0.0068** | **~Rp 110** |

For 200 sessions/day:
- Daily: Rp 22K
- Monthly: Rp 660K

For 1000 sessions/day:
- Daily: Rp 110K
- Monthly: Rp 3.3 juta

This is well within budget.

---

## Performance Targets

End-to-end latency from `submit` to `complete`:
- p50: 25 seconds
- p95: 45 seconds
- p99: 90 seconds (with retries)

If consistently > 60s, investigate:
- LLM API latency (try different model/region)
- FFmpeg performance (CPU bottleneck on cheap VPS)
- Network to OpenAI/OpenRouter

---

## Monitoring This Pipeline

Track per-stage:
- Success rate (>98% target)
- Latency (p50, p95)
- Cost per call
- Error types

Log to `audit_log` table when:
- LLM returns invalid JSON
- Whisper confidence < 0.7
- Moderation flags content
- Pipeline takes >2x typical duration

Manual review queue: 30 random sessions per week, owner reviews scoring quality. If subjective accuracy drops below 80%, retune prompts.
