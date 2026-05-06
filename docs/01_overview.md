# 01 — Product Overview

## What We Are Building

Arena Politika is a web application where Indonesian users practice political debate skills by recording voice arguments and getting AI-powered feedback.

## The MVP User Journey

A user opens the website, registers with email, lands on a dashboard. They click "Mulai Debat" (Start Debate). They see 5-10 topics to choose from (e.g., "Subsidi BBM harus dicabut total"). They pick one, choose their position (PRO or KONTRA), and pick one of 3 AI personas to "debate" against.

The system explains the rules: 60 seconds of voice recording. Click record. Speak their argument. Click stop (or auto-stop at 60s).

The system shows a loading screen ("AI sedang menganalisis..."). Behind the scenes:
1. Audio is uploaded to backend
2. Whisper transcribes it
3. AI Judge (LLM) scores it on 5 dimensions
4. AI Opponent generates a counter-argument (text only for MVP, no voice)
5. A 15-25 second video clip is generated with the user's audio + visual overlays

Finally, the user sees their score (e.g., 76/100), breakdown by dimension, written feedback, the AI's counter-argument, and the auto-generated clip ready to share or download.

That's the entire MVP.

## What Makes This Worth Building

Three things, in order of importance:

**Voice as the primary interface** is rare in productivity/edutainment apps. It mimics real debate, which is what makes it valuable for skill-building.

**Multi-dimensional AI scoring** is the secret sauce. Most "AI feedback" tools give one number. Scoring 5 dimensions (Logika, Data, Emosi, Konsistensi, Retorika) gives users actionable insight into specific weaknesses.

**Auto-generated viral clips** turn every session into potential marketing. Even if users don't always share, the option drives engagement.

## Who It Is For

For MVP testing, the target user is:
- Indonesian, age 20-35
- University student or young professional
- Active on TikTok/Instagram
- Interested in politics, current affairs, or debate
- Tech-savvy enough to use a web app with microphone access

Not for: children (under 17), users who can't speak Bahasa Indonesia fluently, users on shared/public computers without mic privacy.

## Hard Constraints for MVP

The MVP must:
- Run on a single VPS (Hetzner CCX23 or equivalent, ~Rp 1.5 juta/month)
- Cost less than Rp 6 juta/month total at 200 sessions/day
- Be web-only (no mobile app yet)
- Support only Indonesian language (no i18n)
- Handle at most 50 concurrent users (more than enough for testing)

The MVP must NOT:
- Mention real Indonesian politicians, parties, or specific institutions
- Generate content with hate speech, SARA, or content violating UU ITE
- Store voice recordings beyond 24 hours
- Enable any form of P2P or P2A communication outside the structured debate
- Have any payment functionality

## What "Done" Looks Like

The MVP is shipped when:

A new user can land on the site, register, complete one full debate session, see their score, generate a clip, and either download or share it. All within 5 minutes of arrival. Without bugs. With acceptable performance (<3s API response, <30s for full scoring).

Then 30 alpha testers (recruited by the owner) can do the same. We collect feedback for 2 weeks. Iterate on top 3 issues. Then we decide whether to build v2 (party/election system) or pivot.

## Out-of-Scope Reminders

These features are tempting but explicitly NOT in MVP:

The clan/party system is a v2 feature. Building it requires complex permissions, persistent state across many users, and significantly more LLM cost. It must wait.

The election system depends on the party system. Same reason.

The cognitive compass / ideology profiling is a v2 feature. It needs many sessions per user to be meaningful. Useless at MVP scale.

Career progression beyond a basic XP counter is a v2 feature. The 8-rank system from the original blueprint is overkill for testing core mechanic.

Player-vs-player debate is a v2 feature. Significantly harder to moderate and rate-limit.

News engine (auto topic generation from RSS feeds) is included in MVP — see `specs/08_news_engine.md`. Targets 5-10 new topic candidates per day with manual approval. Evergreen seed (10-15 manual topics) provides fallback for day-one launch before the engine produces output.

Anti-cheat with voiceprint analysis is a v2 feature. For MVP, we just rate-limit and reject suspiciously fast/identical submissions.

If during development you (or an AI agent) feels strongly that one of these is needed for MVP — flag it explicitly to the owner. Do not silently expand scope.

## Key Decisions Already Made

These decisions are settled. Do not relitigate:

**Web-first, not mobile-first.** Voice quality is better on desktop microphones. Target user is at home/dorm, not on the go.

**No real-time multiplayer.** Asynchronous (record-then-respond) is technically simpler and produces better debate quality (people think before speaking).

**Single VPS, no cloud.** Anti-fragility for MVP comes from simplicity, not redundancy. We can scale horizontally later.

**Bahasa Indonesia only.** Adding English doubles content moderation complexity for half the user base. Defer.

**OpenRouter, not direct API.** Multi-model fallback is cheap insurance against rate limits and outages.

**Whisper API, not self-hosted.** Self-hosting requires GPU which is overkill for MVP volume.

**No SaaS auth (Auth0, Clerk).** They're great but cost money at scale. Self-hosted JWT is fine for MVP.

## Success Metrics (For Post-MVP Review)

After 30 days of alpha testing, evaluate:

- **Session completion rate**: % of started sessions that produce a final score. Target: > 80%.
- **Day-7 retention**: % of registered users who return within 7 days. Target: > 40%.
- **Sessions per active user per week**: Target median: > 3.
- **Clip share rate**: % of completed sessions where user clicks share. Target: > 25%.
- **Scoring accuracy (manual review)**: 30 random sessions reviewed by owner. % where score "feels right" within ±10 points. Target: > 80%.
- **Total infrastructure cost**: Target: < Rp 6 juta/month at 200 sessions/day.

If these are hit, build v2. If not, fix the gaps before adding features.
