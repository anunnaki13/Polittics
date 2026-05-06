# Spec 03 — AI Personas

## Purpose

Define the 3 AI personas a player can debate against in the MVP. Each persona has a distinct voice, debate style, and difficulty level. The persona behavior is driven entirely by LLM system prompts — no separate logic or fine-tuning.

## Why 3 Personas Only

We considered 8 personas in early design (Si Profesor, Si Pak RT, Si Aktivis 98, Si Influencer, Si Birokrat, Si Ulama Modern, Si Mantan Menteri, Si Founding Father). For MVP, we ship only 3 because:

1. Each persona needs careful prompt tuning. 3 takes ~1 week, 8 takes ~3 weeks.
2. Player needs variety, not paralysis. 3 covers low/medium/high difficulty.
3. Adding more personas post-launch is trivial — just add a row to DB and a prompt.

## The 3 Personas

### Persona 1: Si Pak RT (Easy)
- **Slug**: `si_pak_rt`
- **Archetype**: Populis Akar Rumput
- **Difficulty**: Mudah
- **Target Score**: 60 (player must score >62 to win)
- **Tagline**: "Yang penting masuk akal buat warga"
- **Description**: Bapak RT yang bicara dengan bahasa sederhana, membawa argumen dari pengalaman lapangan dan keseharian warga. Tidak terlalu menggunakan data statistik, lebih ke common sense dan analogi rumah tangga.

**Debate style:**
- Bahasa santai, sering pakai analogi kehidupan sehari-hari
- Mengutip "tetangga saya bilang", "kalau di kampung kami"
- Jarang pakai data statistik
- Mudah ditebak strukturnya
- Cocok untuk pemula

**Why beat-able:** Argumen mudah dilawan dengan data konkret atau logika yang lebih ketat.

---

### Persona 2: Si Aktivis 98 (Medium)
- **Slug**: `si_aktivis_98`
- **Archetype**: Idealis Reformis
- **Difficulty**: Sedang
- **Target Score**: 70 (player must score >72 to win)
- **Tagline**: "Suara rakyat tidak boleh diam"
- **Description**: Mantan aktivis era reformasi 1998. Membawa semangat moral dan keadilan sosial. Argumennya kuat secara emosional dan etis, sering merujuk pada prinsip-prinsip demokrasi dan hak asasi.

**Debate style:**
- Retorika kuat, banyak menggunakan kalimat persuasif
- Argumen berbasis nilai (keadilan, kebenaran, hak rakyat)
- Sering merujuk pada sejarah perjuangan dan prinsip universal
- Emosional tapi terkontrol
- Pertanyaan retoris yang menggugah

**Why beat-able:** Bisa dilawan dengan argumen pragmatis (vs idealistis), atau dengan menunjukkan inkonsistensi internal idealisme.

---

### Persona 3: Si Profesor (Hard)
- **Slug**: `si_profesor`
- **Archetype**: Intelektual Akademis
- **Difficulty**: Sulit
- **Target Score**: 80 (player must score >82 to win)
- **Tagline**: "Data adalah senjata terbaik"
- **Description**: Profesor universitas yang sangat data-driven. Setiap argumen disertai statistik, kutipan jurnal, dan referensi kebijakan internasional. Logikanya rapat, sulit diserang.

**Debate style:**
- Formal dan terstruktur (claim-evidence-warrant)
- Mengutip data BPS, World Bank, jurnal akademis
- Membongkar fallacy logika dengan presisi
- Tidak emosional, sangat tenang
- Memberi counter-argument yang membutuhkan pengetahuan substantif

**Why beat-able:** Sulit, tapi bisa kalau pemain juga punya data dan struktur yang rapat. Sering kali profesor terlalu akademis dan kurang grounded — bisa diserang dengan implementasi praktis.

---

## Prompt Engineering Approach

Each persona has TWO types of prompts:

### Type A: Opponent Response Prompt
Used when generating the AI's counter-argument after player records.
Located in: `prompts/opponent_prompts.md`

### Type B: Scoring Calibration Hints
The persona affects scoring difficulty. Higher-difficulty persona = stricter scoring.
This is implemented inside the scoring prompt with a section like:

```
PERSONA LAWAN: {persona.name}
KETAT_SKORING: {persona.difficulty == 'sulit' ? 'TINGGI' : 'NORMAL'}
```

In `prompts/scoring_prompt.md`, the LLM is instructed to score 5-10% lower against harder personas to reflect the higher debate standard.

## Persona Database Seeding

The 3 personas are seeded into the `personas` table during initial deployment. The seed script reads from `prompts/opponent_prompts.md` and inserts each persona with their full system prompt.

### Seed Script Behavior
```python
# backend/scripts/seed_personas.py
PERSONAS = [
    {
        "slug": "si_pak_rt",
        "name": "Si Pak RT",
        "archetype": "Populis Akar Rumput",
        "description": "...",
        "tagline": "Yang penting masuk akal buat warga",
        "difficulty": "mudah",
        "target_score": 60,
        "system_prompt": load_prompt("opponent_si_pak_rt.txt"),
    },
    # ... 2 more
]

async def seed_personas(db):
    for data in PERSONAS:
        existing = await get_persona_by_slug(db, data["slug"])
        if existing:
            # Update prompt if changed (allow iteration)
            existing.system_prompt = data["system_prompt"]
            existing.target_score = data["target_score"]
        else:
            db.add(Persona(**data))
    await db.commit()
```

Idempotent. Safe to run repeatedly to update prompts.

## Tuning Workflow

When a prompt produces poor responses (LLM goes off-character, too verbose, too short):

1. Identify failure pattern by reviewing 10+ real responses
2. Edit the prompt in `prompts/opponent_prompts.md`
3. Re-run seed script to update DB
4. Test with 5 manual debates per persona
5. Compare quality against previous version
6. If improved, commit. If not, revert.

Version-control the prompt files. Each prompt change should be a separate commit with rationale.

## Persona Selection UX

In the frontend persona selector, each persona card shows:

```
┌─────────────────────────────────┐
│ [Avatar Placeholder]            │
│                                 │
│ Si Pak RT                       │
│ Populis Akar Rumput             │
│                                 │
│ "Yang penting masuk akal        │
│  buat warga"                    │
│                                 │
│ Kesulitan: ⭐⭐☆☆☆ (Mudah)       │
│ Target Skor: 60                 │
│ Reward: +30 XP                  │
└─────────────────────────────────┘
```

Player picks based on confidence. Easy = lower reward, Hard = higher reward.

## Avatar/Visual Identity

For MVP, no actual avatar images needed. Use stylized initials in colored circles:
- Si Pak RT: Brown circle with "PR"
- Si Aktivis 98: Red circle with "A98"
- Si Profesor: Blue circle with "Pr"

Or use simple SVG illustrations (can be drawn by Claude Design later). Defer photorealistic avatars to v2.

## Acceptance Criteria

- [ ] 3 personas exist in DB with correct attributes
- [ ] Each persona has a working system prompt (produces in-character responses)
- [ ] Persona selector shows all 3 with correct visual treatment
- [ ] Selecting persona affects scoring difficulty (verified via test debates)
- [ ] Persona's response stays in character across 10 test runs
- [ ] No persona generates off-topic, offensive, or politically-named content

## Edge Cases

### Persona Generates Long Response
If LLM ignores word limit and returns 300+ words, truncate at sentence boundary near 150 words. Log warning for prompt tuning.

### Persona Mentions Real Politicians/Parties
This is a hard fail. The system prompt explicitly forbids it. If detected, regenerate with stricter prompt. If still occurs, mark debate failed and show generic response.

### Persona Goes Off-Topic
If the response doesn't address the player's argument, regenerate. Max 2 retries. If still off, show fallback static response: "Tanggapan tidak dapat di-generate. Silakan coba lagi."

## Out of Scope for MVP

- Voice synthesis for AI opponent (TTS) — text only for MVP
- Persona "memory" across sessions (each debate is stateless)
- User-created custom personas
- Persona evolution / leveling up
- Multi-persona panel debate (1v3)
