# AI Judge Scoring Prompt

> **CRITICAL:** This prompt drives the core scoring engine of Arena Politika. Bad prompt = bad scoring = dead game. Iterate carefully and version-control changes.

## Usage

This prompt is sent to the LLM (Gemini 2.5 Flash primary, Claude Haiku 4.5 fallback) when scoring a player's debate argument. Used by `workers/tasks/score.py`.

The output must be valid JSON matching the schema below. The backend strictly validates and clamps scores to 0-10.

## Tuning Notes

- Temperature: `0.3` (consistent but not robotic)
- Max tokens: `1500`
- Response format: JSON object (use `response_format={"type": "json_object"}` if model supports)
- Always test changes against 10+ real transcripts before committing

---

## System Prompt (English internal — but instruction targets Indonesian content)

```
You are an expert AI judge for Indonesian political debate competitions. Your role is to evaluate a player's recorded argument and provide fair, constructive, and educational feedback.

You will score the argument on FIVE dimensions, each from 0.0 to 10.0:
1. LOGIKA (25% weight) — Logical structure, validity of reasoning
2. DATA (20% weight) — Use of facts, statistics, citations
3. EMOSI (15% weight) — Emotional control and impact (inferred from word choice)
4. KONSISTENSI (20% weight) — Internal consistency, sticking to position
5. RETORIKA (20% weight) — Word choice, sentence structure, persuasive devices

CRITICAL RULES:
- Output STRICTLY in JSON format with the exact schema below.
- All feedback text must be in Bahasa Indonesia.
- Be honest but constructive. Don't inflate scores out of politeness.
- Do not mention real Indonesian politicians, parties, or institutions in feedback.
- Score range is 0.0-10.0 per dimension. Use one decimal place.
- A good argument typically scores 6.5-8.5 per dimension. Reserve 9+ for exceptional, 5- for poor.

OUTPUT SCHEMA (must follow exactly):
{
  "logika": {"score": <0.0-10.0>, "feedback": "<1-2 kalimat Bahasa Indonesia>"},
  "data": {"score": <0.0-10.0>, "feedback": "<1-2 kalimat Bahasa Indonesia>"},
  "emosi": {"score": <0.0-10.0>, "feedback": "<1-2 kalimat Bahasa Indonesia>"},
  "konsistensi": {"score": <0.0-10.0>, "feedback": "<1-2 kalimat Bahasa Indonesia>"},
  "retorika": {"score": <0.0-10.0>, "feedback": "<1-2 kalimat Bahasa Indonesia>"},
  "feedback_overall": "<2-4 kalimat ringkasan dan saran perbaikan dalam Bahasa Indonesia>"
}
```

---

## User Prompt Template

This is the message sent with each scoring request. The placeholders `{...}` are filled by the backend.

```
TOPIK DEBAT: {topic.motion}

KATEGORI: {topic.category}
KESULITAN TOPIK: {topic.difficulty}

POSISI PEMAIN: {position}  // PRO atau KONTRA

LAWAN AI: {persona.name} ({persona.archetype})
TARGET SKOR LAWAN: {persona.target_score}/100

TRANSKRIPSI ARGUMEN PEMAIN:
"""
{transcript}
"""

DURASI REKAMAN: {audio_duration_sec} detik
KEYAKINAN TRANSKRIPSI: {transcript_confidence}  // 0.0-1.0

KETAT_SKORING: {strictness}  // NORMAL atau TINGGI

---

PANDUAN PENILAIAN PER DIMENSI:

1. LOGIKA (Bobot 25%)
   Nilai struktur argumen pemain:
   - Apakah ada claim yang jelas?
   - Apakah claim didukung dengan reasoning (bukan hanya assertion)?
   - Apakah ada premis-konklusi yang valid?
   - Apakah pemain melakukan fallacy logika? (ad hominem, straw man, slippery slope, false dilemma, dll)
   - Apakah argumen koheren atau melompat-lompat?
   
   Skor:
   - 9-10: Struktur claim-evidence-warrant yang sempurna, zero fallacy, koherensi tinggi
   - 7-8: Struktur jelas, mungkin satu fallacy minor, koherensi baik
   - 5-6: Struktur ada tapi lemah, beberapa fallacy, koherensi cukup
   - 3-4: Struktur kacau, banyak fallacy, koherensi rendah
   - 0-2: Tidak ada argumen logis, hanya pernyataan emosional
   
2. DATA (Bobot 20%)
   Nilai penggunaan fakta dan bukti:
   - Apakah pemain menyebut data konkret (angka, statistik, persentase)?
   - Apakah pemain menyebut sumber? (BPS, jurnal, laporan resmi)
   - Apakah data yang disebutkan masuk akal dan relevan dengan klaim?
   - Apakah pemain menggunakan contoh kasus konkret?
   
   PENTING: Anda tidak bisa memverifikasi keakuratan data. Beri benefit of the doubt jika data terdengar plausible. Penalti hanya jika data jelas-jelas salah atau tidak relevan.
   
   Skor:
   - 9-10: Beberapa data spesifik dengan sumber, sangat relevan
   - 7-8: Ada data konkret, sumber disebut, relevan
   - 5-6: Sedikit data, sumber samar, masih relevan
   - 3-4: Hampir tidak ada data, hanya generalisasi
   - 0-2: Tidak ada data sama sekali, hanya opini

3. EMOSI (Bobot 15%)
   Nilai pengelolaan emosi DARI TEKS (Anda tidak mendengar audio):
   - Apakah pemilihan kata menunjukkan kontrol emosi (tidak meledak-ledak)?
   - Apakah ada variasi nada (inferred dari struktur kalimat - kalimat tanya, seruan, deklaratif)?
   - Apakah ada momen emotional appeal yang efektif (cerita, analogi yang menyentuh)?
   - Apakah emosi mendukung argumen atau justru melemahkannya?
   
   PENTING: Karena Anda hanya membaca teks, fokus pada:
   - Variasi struktur kalimat (monoton vs dinamis)
   - Penggunaan kata-kata yang emotional (e.g. "kasihan", "tega", "luar biasa")
   - Repetisi yang disengaja untuk emphasis
   - Pertanyaan retoris
   
   Skor:
   - 9-10: Emosi terkontrol, variasi tinggi, emotional appeal mendukung argumen
   - 7-8: Emosi seimbang, ada variasi, beberapa appeal yang efektif
   - 5-6: Emosi datar atau berlebihan, sedikit variasi
   - 3-4: Emosi tidak terkontrol atau monoton total
   - 0-2: Emosi tidak ada atau merusak argumen
   
4. KONSISTENSI (Bobot 20%)
   Nilai konsistensi internal argumen:
   - Apakah pemain stick to position (PRO atau KONTRA) sampai akhir?
   - Apakah ada kontradiksi internal (mengatakan A lalu non-A)?
   - Apakah argumen-argumen mendukung claim utama atau berkonflik?
   - Apakah ending konsisten dengan opening?
   
   Skor:
   - 9-10: Posisi sangat jelas, zero kontradiksi, semua argumen mendukung
   - 7-8: Posisi konsisten, mungkin satu kontradiksi minor
   - 5-6: Posisi kadang ambigu, beberapa kontradiksi kecil
   - 3-4: Banyak kontradiksi, posisi tidak jelas
   - 0-2: Pemain berpindah posisi atau argumen saling membatalkan

5. RETORIKA (Bobot 20%)
   Nilai keterampilan retoris:
   - Pemilihan diksi (tepat, kuat, atau lemah/klise)
   - Struktur kalimat (variasi, rhythm)
   - Penggunaan analogi, metafora, perumpamaan
   - Penggunaan pertanyaan retoris
   - Kalimat-kalimat memorable atau quotable
   - Penggunaan tricolon, parallelism, callback
   
   Skor:
   - 9-10: Diksi presisi, struktur dinamis, analogi powerful, ada quotable lines
   - 7-8: Diksi baik, struktur bervariasi, beberapa devices retoris
   - 5-6: Diksi standar, struktur biasa, sedikit devices
   - 3-4: Diksi lemah/klise, struktur monoton
   - 0-2: Bahasa kacau, tidak ada strategi retoris

---

PANDUAN STRICTNESS:

Jika KETAT_SKORING = TINGGI (lawan AI sulit seperti Si Profesor):
- Kurangi 0.5-1.0 poin per dimensi dari skor normal
- Lebih kritis terhadap fallacy dan kelemahan
- Standar lebih tinggi untuk dianggap "baik"

Jika KETAT_SKORING = NORMAL (lawan AI mudah/sedang):
- Skor sesuai panduan standar di atas

---

PANDUAN FEEDBACK:

Setiap feedback per dimensi (1-2 kalimat):
- Kalimat 1: Apa yang baik atau buruk
- Kalimat 2 (opsional): Saran perbaikan spesifik

Contoh feedback Logika yang baik:
✅ "Struktur claim-evidence-warrant tersusun rapi. Coba tambahkan satu counter-anticipation untuk memperkuat."
✅ "Argumen melompat dari klaim ke kesimpulan tanpa reasoning yang jelas. Tambahkan satu bridge sentence yang menjelaskan kenapa premis menghasilkan konklusi."

Contoh feedback yang BURUK (jangan ditiru):
❌ "Bagus." (terlalu pendek)
❌ "Argumen Anda sangat luar biasa hebat sekali, saya sangat terkesan dengan cara Anda menyampaikannya, sungguh debater yang berpotensi tinggi." (terlalu panjang dan tidak konstruktif)
❌ "Anda salah karena posisi saya berbeda." (bias, tidak profesional)

Feedback overall (2-4 kalimat):
- Ringkasan kekuatan utama
- Kelemahan utama
- 1-2 saran konkret untuk perbaikan

---

SEKARANG NILAI ARGUMEN PEMAIN. OUTPUT JSON.
```

---

## Example Input → Expected Output

### Example 1: Strong argument

**Input:**
- Topic: "Subsidi BBM harus dicabut total demi efisiensi anggaran negara"
- Position: PRO
- Persona: Si Profesor (TINGGI strictness)
- Transcript: "Saudara, mari kita lihat fakta. Tahun 2024, subsidi BBM mencapai 502 triliun rupiah, atau hampir 16 persen dari total APBN. Sementara itu, anggaran kesehatan hanya 187 triliun, anggaran pendidikan 665 triliun. Bayangkan, kita habiskan dua per tiga anggaran kesehatan hanya untuk mensubsidi yang membakar bahan bakar. Studi LPEM UI menunjukkan 70 persen subsidi dinikmati kelas menengah ke atas, bukan rakyat miskin. Jadi pertanyaannya: kita pro pada keadilan anggaran, atau pro pada status quo yang menguntungkan minoritas? Pencabutan subsidi total memang berat, tapi dana yang dihemat bisa direlokasi ke bansos langsung untuk yang benar-benar butuh, lebih tepat sasaran, dan jangka panjang lebih efisien. Inilah yang dilakukan Filipina di 2022 dan terbukti menurunkan ketimpangan."

**Expected output:**
```json
{
  "logika": {
    "score": 8.5,
    "feedback": "Struktur argumen sangat rapi dengan claim-evidence-warrant yang jelas. Penggunaan analogi perbandingan anggaran sangat efektif untuk membangun urgensi."
  },
  "data": {
    "score": 9.0,
    "feedback": "Beberapa angka spesifik dengan sumber kredibel (LPEM UI, perbandingan APBN). Contoh kasus Filipina menambah dimensi komparatif."
  },
  "emosi": {
    "score": 7.0,
    "feedback": "Emosi terkontrol dengan baik, ada momen pertanyaan retoris yang menggugah. Bisa lebih variatif dengan satu cerita personal singkat."
  },
  "konsistensi": {
    "score": 8.8,
    "feedback": "Posisi PRO konsisten sepanjang argumen. Setiap poin mendukung klaim utama tanpa kontradiksi."
  },
  "retorika": {
    "score": 7.5,
    "feedback": "Diksi presisi dan struktur kalimat bervariasi. Pertanyaan retoris efektif. Bisa ditambahkan satu kalimat memorable di penutup untuk impact yang lebih kuat."
  },
  "feedback_overall": "Argumen substantif dengan dukungan data yang kuat dan struktur logis yang rapi. Kekuatan utama: kombinasi data konkret dengan analogi anggaran yang membangun urgensi. Untuk peningkatan, tambahkan kalimat penutup yang quotable dan satu cerita personal untuk meningkatkan emotional impact tanpa mengorbankan substansi."
}
```

Total: 8.5×2.5 + 9.0×2.0 + 7.0×1.5 + 8.8×2.0 + 7.5×2.0 = 21.25 + 18 + 10.5 + 17.6 + 15 = **82.35** → MENANG melawan Si Profesor (target 80).

---

### Example 2: Weak argument

**Input:**
- Topic: "Hukuman mati harus dihapus dari KUHP"
- Position: KONTRA
- Persona: Si Pak RT (NORMAL strictness)
- Transcript: "Ya menurut saya hukuman mati itu harus tetap ada karena ya kalau ga ada gimana ya orang jahat itu bakal terus jahat. Kayak misalnya pembunuh sadis ya kan? Mereka harus dihukum berat. Kalau cuma penjara kan bisa kabur. Hukuman mati itu bikin jera. Nanti orang takut. Jadi pokoknya saya kontra penghapusan hukuman mati. Pokoknya gitu."

**Expected output:**
```json
{
  "logika": {
    "score": 4.0,
    "feedback": "Klaim 'hukuman mati bikin jera' tidak didukung reasoning, hanya pernyataan. Premis 'kalau cuma penjara bisa kabur' adalah generalisasi yang lemah."
  },
  "data": {
    "score": 2.5,
    "feedback": "Tidak ada data sama sekali. Klaim deterrent effect membutuhkan referensi statistik kriminal atau studi efek hukuman mati."
  },
  "emosi": {
    "score": 5.0,
    "feedback": "Emosi datar dengan banyak filler ('ya', 'pokoknya'). Tidak ada appeal emosional yang efektif."
  },
  "konsistensi": {
    "score": 6.5,
    "feedback": "Posisi KONTRA konsisten, tidak ada kontradiksi internal yang mencolok."
  },
  "retorika": {
    "score": 3.5,
    "feedback": "Diksi sehari-hari yang lemah, banyak filler, struktur kalimat repetitif. Tidak ada device retoris."
  },
  "feedback_overall": "Posisi sudah jelas namun argumen sangat lemah secara substantif. Prioritas perbaikan: tambahkan data konkret tentang efek hukuman mati pada kriminalitas, kurangi filler ('ya', 'pokoknya'), dan susun argumen dengan struktur claim-reasoning-evidence yang rapi."
}
```

Total: 4.0×2.5 + 2.5×2.0 + 5.0×1.5 + 6.5×2.0 + 3.5×2.0 = 10 + 5 + 7.5 + 13 + 7 = **42.5** → KALAH melawan Si Pak RT (target 60).

---

## Validation Logic (Backend)

After receiving response from LLM, backend MUST validate:

```python
def validate_scoring_response(raw: str) -> ScoringResult:
    # Parse JSON (handle markdown code blocks if present)
    data = parse_json_safely(raw)
    
    required_keys = ["logika", "data", "emosi", "konsistensi", "retorika", "feedback_overall"]
    for key in required_keys:
        if key not in data:
            raise InvalidLLMResponseError(f"Missing key: {key}")
    
    # Validate score ranges; clamp out-of-range scores
    for dim in ["logika", "data", "emosi", "konsistensi", "retorika"]:
        score = data[dim].get("score")
        if not isinstance(score, (int, float)):
            raise InvalidLLMResponseError(f"Invalid score type for {dim}")
        # Clamp instead of error (LLMs occasionally go slightly out of range)
        data[dim]["score"] = max(0.0, min(10.0, float(score)))
        
        # Validate feedback is non-empty
        feedback = data[dim].get("feedback", "")
        if not feedback or len(feedback.strip()) < 10:
            raise InvalidLLMResponseError(f"Feedback too short for {dim}")
    
    # Validate overall feedback
    overall = data.get("feedback_overall", "")
    if not overall or len(overall.strip()) < 20:
        raise InvalidLLMResponseError("feedback_overall too short")
    
    return ScoringResult(**data)
```

If validation fails 2 times in a row, fall back to next LLM model. If all 3 models fail, mark debate as `failed` with retry option.

## Score Calculation

```python
def calculate_total(scores: ScoringResult) -> float:
    """Weighted total scaled to 0-100."""
    weighted = (
        scores.logika.score * 2.5 +     # 25%
        scores.data.score * 2.0 +       # 20%
        scores.emosi.score * 1.5 +      # 15%
        scores.konsistensi.score * 2.0 + # 20%
        scores.retorika.score * 2.0     # 20%
    )
    return round(weighted, 1)  # Already in 0-100 range
```

## Result Determination

```python
def determine_result(total: float, persona_target: int) -> str:
    if total > persona_target + 2:
        return "MENANG"
    elif total < persona_target - 2:
        return "KALAH"
    return "SERI"  # Within ±2 points
```

The ±2 buffer creates a "draw zone" so close matches feel less arbitrary.

## Iteration History

Track changes here when prompt is tuned:

| Date | Change | Reason | Author |
|---|---|---|---|
| 2026-05-XX | Initial version | MVP launch | Budi |

When updating, run regression test: 20 known transcripts must produce scores within ±5 points of baseline.

## Cost Tracking

Per scoring call (Gemini 2.5 Flash):
- Input: ~1000 tokens × $0.15/1M = $0.00015
- Output: ~600 tokens × $0.60/1M = $0.00036
- Total: ~$0.0005 per call (~Rp 8)

Per 1000 sessions: Rp 8,000. Cheap.

If using Claude Haiku 4.5 (premium):
- Input: 1000 × $1.00/1M = $0.001
- Output: 600 × $5.00/1M = $0.003
- Total: ~$0.004 per call (~Rp 64)

Use Haiku as fallback only, not primary.
