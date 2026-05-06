# Spec 06 — Clip Generator

## Purpose

Auto-generate a 15-25 second highlight video from each completed debate. The clip combines the player's audio with stylized visuals, suitable for sharing on TikTok, Instagram Reels, and WhatsApp.

The clip is the **viral marketing engine** of the product. Every clip shared = potential new user.

## Output Specifications

| Property | Value |
|---|---|
| Resolution | 1080×1920 (vertical, 9:16) |
| Duration | 15-25 seconds (target: 18s) |
| Video codec | H.264 (libx264), preset `fast`, CRF 23 |
| Audio codec | AAC, 128 kbps, 44.1 kHz |
| Container | MP4 |
| Frame rate | 30 fps |
| File size | 1.5-3 MB typical |
| Audio source | Player's recording (segment) |

Vertical format chosen because TikTok/Reels are the primary share targets. Plays correctly when downloaded for stories.

## Visual Layout

```
┌─────────────────────────────┐  ← top of frame (1080×1920)
│                             │
│   [TOPIC banner, scrolling] │  ← top 15% of frame
│   "Subsidi BBM harus..."    │
│                             │
├─────────────────────────────┤
│                             │
│   ╔═══════════════════╗     │
│   ║                   ║     │
│   ║   PLAYER AVATAR   ║     │  ← center 30%
│   ║   (initials in    ║     │
│   ║    colored circle) ║    │
│   ║                   ║     │
│   ╚═══════════════════╝     │
│                             │
│   POSISI: PRO               │
│   vs                        │
│   Si Profesor               │
│                             │
├─────────────────────────────┤
│                             │
│  ▁▃▅▇▇▅▃▁▃▅▇▅▃▁▃▅▇         │  ← waveform 20%
│                             │
├─────────────────────────────┤
│                             │
│   ╔══════════════════╗      │  ← score reveal 25%
│   ║                  ║      │
│   ║      SKOR        ║      │
│   ║       78         ║      │  ← shows in last 3s
│   ║                  ║      │
│   ║     MENANG       ║      │
│   ╚══════════════════╝      │
│                             │
│   arenapolitika.id          │  ← watermark bottom
│                             │
└─────────────────────────────┘
```

## Visual Style

For MVP, simple but polished:
- Background: Dark navy gradient (#050816 → #0a1230)
- Primary accent: Cyan (#00d9ff)
- Secondary accent: Magenta (#ff4477)
- Win badge: Green (#00ff88)
- Lose badge: Red (#ff4444)
- Tie badge: Yellow (#ffaa00)
- Font: Inter Bold for headers, JetBrains Mono for numbers
- Watermark: 15% opacity white text

Aesthetic match: "Bloomberg Terminal meets Esports Arena" — same as the rest of the app.

## Implementation Approach

### FFmpeg Pipeline (Single Pass)

```python
# workers/tasks/clip.py
async def generate_clip(debate_id: UUID):
    debate = await get_debate_with_score(debate_id)
    
    # Step 1: Determine segment (which part of audio to use)
    segment = select_segment(debate.audio_duration_sec, debate.transcript)
    
    # Step 2: Download audio from MinIO to /tmp
    audio_path = await download_audio_to_tmp(debate.audio_url)
    
    # Step 3: Generate waveform image from audio
    waveform_path = generate_waveform_image(
        audio_path,
        start=segment.start,
        duration=segment.duration,
        output_path=f"/tmp/wave_{debate_id}.png",
    )
    
    # Step 4: Render video with FFmpeg
    output_path = f"/tmp/clip_{debate_id}.mp4"
    render_video(
        background=BACKGROUND_TEMPLATE,
        topic_text=truncate(debate.topic.motion, 80),
        position=debate.position,
        persona_name=debate.persona.name,
        player_initials=get_initials(debate.user.display_name),
        waveform=waveform_path,
        score=int(debate.scores.total),
        result=debate.scores.result,  # MENANG/KALAH/SERI
        audio_path=audio_path,
        audio_start=segment.start,
        audio_duration=segment.duration,
        output=output_path,
    )
    
    # Step 5: Upload to MinIO
    clip_url = await upload_clip_to_minio(output_path, debate_id)
    
    # Step 6: Save URL, cleanup temp files
    await save_clip_url(debate_id, clip_url)
    cleanup_temp_files([audio_path, waveform_path, output_path])
```

### Segment Selection

For MVP, simple heuristic:

```python
def select_segment(audio_duration: float, transcript: str) -> Segment:
    """Pick the most quotable segment for the highlight."""
    # If audio is short enough, use whole thing
    if audio_duration <= 25:
        return Segment(start=0, duration=audio_duration)
    
    # Otherwise, take last 18 seconds
    # Reason: closing arguments tend to be most impactful
    # In v2, use LLM to identify the best sentence
    return Segment(start=audio_duration - 18, duration=18)
```

In v2, use LLM analysis:
```
Given this transcript with timestamps, identify the 15-20 second 
segment that contains the most powerful claim or memorable phrase.
```

### Waveform Generation

Use `librosa` (Python) or `ffmpeg`'s built-in `showwavespic`:

```bash
ffmpeg -i input.wav -filter_complex \
  "[0:a]aformat=channel_layouts=mono,compand,showwavespic=s=1080x300:colors=#00d9ff" \
  -frames:v 1 waveform.png
```

This gives a static image that we overlay during the entire clip. For animated waveform (more impressive), use `showwaves` filter — but it's CPU-heavier. Defer to v2.

### Video Rendering

Building a complex FFmpeg command is tedious. Use Python `ffmpeg-python` library for cleaner code:

```python
def render_video(
    background, topic_text, position, persona_name, player_initials,
    waveform, score, result, audio_path, audio_start, audio_duration, output,
):
    # Inputs
    bg = ffmpeg.input(background, loop=1, t=audio_duration)
    audio = ffmpeg.input(audio_path, ss=audio_start, t=audio_duration)
    wave_img = ffmpeg.input(waveform, loop=1, t=audio_duration)
    
    # Color for result badge
    result_color = {"MENANG": "0x00ff88", "KALAH": "0xff4444", "SERI": "0xffaa00"}[result]
    
    # Build video filter chain
    video = bg
    
    # Overlay topic text at top
    video = video.filter(
        'drawtext',
        text=topic_text,
        fontfile=FONT_INTER_BOLD,
        fontsize=42, fontcolor='white',
        x='(w-text_w)/2', y=120,
        line_spacing=10,
        box=1, boxcolor='black@0.4', boxborderw=20,
    )
    
    # Overlay player avatar (circle with initials)
    # For MVP, render this as pre-made PNG per user (or generic colored circle)
    # ... 
    
    # Overlay position + persona text
    video = video.filter(
        'drawtext',
        text=f"POSISI: {position}",
        fontfile=FONT_INTER_BOLD,
        fontsize=36, fontcolor='cyan',
        x='(w-text_w)/2', y=900,
    )
    video = video.filter(
        'drawtext',
        text=f"vs {persona_name}",
        fontfile=FONT_INTER_BOLD,
        fontsize=32, fontcolor='white',
        x='(w-text_w)/2', y=960,
    )
    
    # Overlay waveform
    video = ffmpeg.overlay(video, wave_img, x=0, y=1100)
    
    # Overlay score reveal in last 3 seconds
    video = video.filter(
        'drawtext',
        text='SKOR',
        fontfile=FONT_INTER_BOLD,
        fontsize=48, fontcolor='white',
        x='(w-text_w)/2', y=1450,
        enable=f'gte(t,{audio_duration - 3})',
    )
    video = video.filter(
        'drawtext',
        text=str(score),
        fontfile=FONT_JETBRAINS_BOLD,
        fontsize=200, fontcolor='cyan',
        x='(w-text_w)/2', y=1500,
        enable=f'gte(t,{audio_duration - 3})',
    )
    video = video.filter(
        'drawtext',
        text=result,
        fontfile=FONT_INTER_BOLD,
        fontsize=64, fontcolor=result_color,
        x='(w-text_w)/2', y=1730,
        enable=f'gte(t,{audio_duration - 3})',
        box=1, boxcolor='black@0.6', boxborderw=15,
    )
    
    # Watermark
    video = video.filter(
        'drawtext',
        text='arenapolitika.id',
        fontfile=FONT_INTER_BOLD,
        fontsize=28, fontcolor='white@0.6',
        x='(w-text_w)/2', y=1850,
    )
    
    # Mux video + audio, export
    out = ffmpeg.output(
        video, audio, output,
        vcodec='libx264', preset='fast', crf=23,
        acodec='aac', audio_bitrate='128k',
        pix_fmt='yuv420p',
        movflags='+faststart',  # Enables progressive download
        r=30,
    )
    
    ffmpeg.run(out, overwrite_output=True, quiet=True)
```

### Asset Files

Store these in `backend/assets/` and copy into Docker image:

```
backend/assets/
├── fonts/
│   ├── Inter-Bold.ttf
│   └── JetBrainsMono-Bold.ttf
├── backgrounds/
│   ├── default.png       # 1080x1920 dark navy gradient
│   └── win_overlay.png   # Optional victory glow
└── watermark.png         # Small transparent logo
```

These are part of the repo. Total <2MB.

## Performance

Target render time: <10 seconds for an 18s clip on 4-core CPU.

Profile:
- FFmpeg encoding: ~6s
- Waveform generation: ~1s
- Asset loading + IO: ~2s
- Upload to MinIO: ~1s

If render exceeds 20s consistently, investigate:
- VPS CPU saturation (run during off-peak)
- Use `preset=ultrafast` instead of `fast` (slightly larger files but faster encode)
- Reduce resolution to 720×1280 if needed (still good for social)

## Storage

Clips stored in MinIO bucket `clips/` indefinitely. Naming: `clips/{debate_id}.mp4`.

For privacy, if user deletes account, all their clips are deleted as part of account deletion process.

Storage estimate:
- 200 sessions/day × 2.5 MB avg = 500 MB/day
- Per month: ~15 GB
- Per year: ~180 GB

VPS with 200 GB SSD can hold ~13 months of clips. Plenty for MVP. Migrate older clips to cheaper storage (Backblaze B2) at 6-month mark if needed.

## Failure Handling

If clip generation fails:
1. Retry once with simpler config (no waveform, plain background)
2. If still fails, mark debate `complete` anyway but with `clip_url=null`
3. Show user: "Klip tidak dapat dibuat. Hasil debat tetap tersedia."
4. Log error for investigation

The score and feedback are more important than the clip. Don't fail the whole debate if only clip fails.

## Acceptance Criteria

- [ ] Clip generated within 15s of scoring complete
- [ ] Output is 1080×1920 vertical MP4
- [ ] Duration is 15-25 seconds
- [ ] Audio quality matches original recording
- [ ] All visual elements render correctly (topic, score, result, watermark)
- [ ] Score number is large and readable
- [ ] Result badge has correct color
- [ ] Plays correctly in Chrome, Safari, mobile browsers
- [ ] Plays correctly when downloaded and uploaded to TikTok
- [ ] File size under 4 MB
- [ ] Watermark visible but not obtrusive

## Out of Scope for MVP

- LLM-based segment selection (use heuristic for MVP)
- Animated waveform (use static image)
- Custom themes per persona
- Multiple aspect ratios (16:9 horizontal, 1:1 square)
- Subtitle overlay
- Background music
- Video transitions / animations
- User-customizable templates
- Audio enhancement (noise reduction, normalization beyond compand)
- Per-user video signatures (e.g., custom outro)
