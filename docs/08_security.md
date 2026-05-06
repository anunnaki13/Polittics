# 08 — Security

## Threat Model

For MVP, primary threats:
1. **Account compromise** — Stolen credentials, brute force
2. **Spam/abuse** — Bots creating accounts, automated submissions
3. **Cost attacks** — Malicious users running up LLM bills
4. **Data leaks** — Voice recordings, PII exposure
5. **Injection attacks** — SQL injection, XSS, malicious LLM prompts
6. **Content abuse** — Hate speech, illegal content via voice

Not in MVP threat model (defer):
- Sophisticated targeted attacks
- DDoS at scale
- Insider threats
- Compliance with specific regulations (GDPR, HIPAA, etc.)

## Authentication

### Password Storage
- Hash: Argon2id with default Argon2-cffi parameters
- Never: MD5, SHA-1, plaintext, or even bcrypt (Argon2 is current best practice)

```python
from argon2 import PasswordHasher
ph = PasswordHasher()

# Register
hashed = ph.hash(plain_password)

# Login
try:
    ph.verify(stored_hash, plain_password)
    if ph.check_needs_rehash(stored_hash):
        # Rehash with current params
        new_hash = ph.hash(plain_password)
        await update_password_hash(user_id, new_hash)
except VerifyMismatchError:
    raise InvalidCredentialsError()
```

### Password Requirements
- Minimum 8 characters
- At least 1 letter and 1 number
- Maximum 128 characters (prevents DoS via huge passwords)
- No requirement for special chars (security theater; long passwords matter more)

Don't enforce:
- Mandatory password rotation (NIST 2024 says don't)
- Forbidden character lists (just allow everything)
- Password history (overkill for MVP)

### JWT Tokens

Two tokens:
- **Access token**: 15 minutes expiry, used in `Authorization: Bearer` header
- **Refresh token**: 7 days expiry, stored in HTTPOnly cookie

Access token claims:
```json
{
  "sub": "123",          // user_id
  "username": "budi_pku",
  "iat": 1714989600,
  "exp": 1714990500,
  "type": "access"
}
```

Refresh token claims (minimal):
```json
{
  "sub": "123",
  "iat": 1714989600,
  "exp": 1715594400,
  "type": "refresh",
  "jti": "uuid"          // For revocation tracking
}
```

When refresh token is used, issue new access + new refresh (rotation). Add old refresh `jti` to Redis blacklist with TTL = remaining lifetime.

### Logout
- Add refresh token `jti` to blacklist
- Frontend deletes access token from memory
- Cookie cleared on response

### Brute Force Protection
- 5 failed login attempts per IP in 15 minutes → 15 min lockout
- 3 failed login attempts per email in 15 minutes → 15 min lockout
- Both lockouts stack
- Use Redis with sliding window counter

```python
async def check_login_rate_limit(ip: str, email: str):
    ip_key = f"login:ip:{ip}"
    email_key = f"login:email:{email}"
    
    ip_count = await redis.incr(ip_key)
    if ip_count == 1:
        await redis.expire(ip_key, 900)  # 15 min
    if ip_count > 5:
        raise RateLimitError("Too many login attempts from this IP")
    
    email_count = await redis.incr(email_key)
    if email_count == 1:
        await redis.expire(email_key, 900)
    if email_count > 3:
        raise RateLimitError("Too many login attempts for this account")
```

On successful login, reset counters.

---

## Authorization

### Resource Ownership
Every resource (debate, score, clip) is owned by a single user. Cross-user access is forbidden.

```python
async def get_debate_for_user(db, debate_id, user_id):
    stmt = select(Debate).where(
        Debate.id == debate_id,
        Debate.user_id == user_id  # Always check ownership
    )
    debate = await db.execute(stmt).scalar_one_or_none()
    if not debate:
        raise NotFoundError()  # Don't reveal whether it exists for other user
    return debate
```

### Admin
For MVP, admin operations (banning users, deleting content) are done via direct DB access by the owner. No admin UI yet.

---

## Input Validation

### Pydantic on Backend
All request bodies validated by Pydantic. No raw dict passing.

```python
class DebateCreate(BaseModel):
    topic_id: UUID
    persona_id: UUID
    position: Literal["PRO", "KONTRA"]
    
    # No additional fields allowed
    model_config = ConfigDict(extra="forbid")
```

### Zod on Frontend
Frontend validates before sending to API. Defense in depth.

### File Upload Validation
For audio uploads:
- Max size: 5 MB (enforced by MinIO presigned URL params)
- Allowed MIME types: `audio/webm`, `audio/ogg`, `audio/mp4`
- Max duration: 65 seconds (5s buffer over the 60s soft limit)
- Validated server-side after upload (FFprobe to check metadata)

```python
async def validate_audio_file(file_path: Path) -> AudioMeta:
    probe = ffmpeg.probe(str(file_path))
    duration = float(probe['format']['duration'])
    if duration > 65:
        raise ValidationError("Audio too long")
    
    streams = probe['streams']
    audio_stream = next((s for s in streams if s['codec_type'] == 'audio'), None)
    if not audio_stream:
        raise ValidationError("No audio stream")
    
    if audio_stream['codec_name'] not in ['opus', 'aac', 'mp3']:
        raise ValidationError(f"Unsupported codec: {audio_stream['codec_name']}")
    
    return AudioMeta(duration=duration, codec=audio_stream['codec_name'])
```

If invalid, delete uploaded file and reject debate submission.

---

## SQL Injection Prevention

- Always use SQLAlchemy ORM with parameterized queries
- Never construct SQL from string concatenation
- Never expose raw SQL to user input

```python
# Good
stmt = select(User).where(User.email == email)

# BAD - Don't do this
stmt = text(f"SELECT * FROM users WHERE email = '{email}'")
```

The codebase has zero raw SQL except in migrations (which are static).

---

## XSS Prevention

- React escapes by default — never use `dangerouslySetInnerHTML` for user content
- Sanitize transcript before display? No, React handles it.
- Audit any place using `innerHTML` or similar manual DOM manipulation

LLM-generated content (opponent responses, feedback): treat as untrusted, render via React's normal text rendering.

---

## CORS

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.APP_CORS_ORIGINS.split(","),  # Whitelist only
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)
```

Never `allow_origins=["*"]` in production.

---

## Rate Limiting

### IP-Based (in Caddy)
- 30 requests/minute per IP for general API endpoints
- Caddy returns 429 with `Retry-After` header

### User-Based (in FastAPI middleware)
Per user, per day:
- Debate creation: 10/day (free tier)
- Status polling: 60/min (allows aggressive polling during active session)
- All other endpoints: 60/min

Implementation:
```python
async def rate_limit_middleware(request: Request, call_next):
    user_id = extract_user_id_from_jwt(request)
    if not user_id:
        return await call_next(request)
    
    endpoint = request.url.path
    limit_key = get_limit_key(endpoint, user_id)
    
    # Sliding window via Redis
    count = await redis.incr(limit_key)
    if count == 1:
        await redis.expire(limit_key, get_window(endpoint))
    
    if count > get_limit(endpoint):
        return JSONResponse(
            status_code=429,
            content={"detail": "Rate limit exceeded"},
            headers={"Retry-After": str(await redis.ttl(limit_key))}
        )
    
    return await call_next(request)
```

### Cost-Based Limit (Defensive)
If a single user accumulates >$1 in LLM costs in 24h (~200 sessions), auto-suspend their account and alert admin. Prevents runaway costs from compromised account or abuse.

---

## Content Moderation

### Pre-Scoring Moderation
After Whisper transcription, before LLM scoring:

```python
async def moderate_transcript(transcript: str) -> ModerationResult:
    response = await llm_call(
        model="google/gemini-2.5-flash",
        system="""You are a content moderator. Identify if the text contains:
1. SARA hate speech (racial, religious, ethnic, group-based)
2. Real Indonesian politician names
3. Real Indonesian political party names
4. Defamation
5. Sexual content
6. Violence threats

Respond with JSON: {"flagged": bool, "categories": [list], "severity": "low"/"medium"/"high"}""",
        user=transcript,
        max_tokens=200,
        temperature=0.0,
    )
    return parse_moderation(response)
```

### Action Based on Severity
- **Low** (e.g., mentioning a politician once): Allow, but warn user, log to audit
- **Medium** (e.g., mild defamation): Allow, score reduced 20%, warn user, log
- **High** (e.g., hate speech): Reject, mark debate failed, increment violation counter

### Three-Strike Policy
If user accumulates 3 high-severity violations in 30 days, auto-suspend account pending owner review.

### Sharing Moderation
Before generating clip, run additional check on transcript. If flagged at any level, do not generate clip (only show score privately).

---

## Privacy

### Voice Recording Lifecycle
1. User records audio (browser)
2. Uploaded to MinIO with 24h lifecycle policy
3. Whisper transcribes (transcript saved to DB)
4. Audio used for clip generation
5. After clip is generated, audio is deleted (via cleanup task)
6. Even if cleanup fails, MinIO 24h lifecycle deletes it

The user's voice file never persists more than 24 hours, even in worst case.

### Data User Can Request
- Their account info: `GET /api/auth/me`
- Their debate history: `GET /api/debates`
- Their full debate details: `GET /api/debates/{id}`

### Data Deletion
For MVP, deletion is manual: user emails owner, owner deletes from DB. Add self-service deletion in v2.

When deleting:
1. Delete user's clips from MinIO
2. Delete user's `scores` rows
3. Delete user's `debates` rows
4. Delete user's `rate_limits` rows
5. Anonymize `audit_log` entries (keep for security but remove user_id)
6. Delete user's `users` row

### IP Address Storage
- Never store plaintext IPs
- For rate limiting, use IP only in Redis (auto-expires)
- For audit log, store SHA-256 hash of IP

---

## Secret Management

### `.env.prod` File
- Never commit to git
- File permissions: `chmod 600` (owner read only)
- Rotate secrets quarterly

### Secrets Rotation Process
1. Generate new secret (`openssl rand -hex 32`)
2. Update `.env.prod` on VPS
3. Restart relevant services (`docker compose restart backend worker`)
4. For JWT_SECRET rotation: brief logout-all impact (acceptable)

### API Keys
- OpenRouter: View usage daily, alert if anomaly
- OpenAI (Whisper): Same
- Both should have separate keys for dev/prod
- Both should have monthly spending caps configured

---

## HTTPS

### Caddy Auto-HTTPS
Caddy automatically:
- Provisions Let's Encrypt certs
- Renews them
- Redirects HTTP → HTTPS
- Configures HSTS

No manual cert management. Just point DNS at Caddy.

### TLS Configuration
Caddy uses modern defaults:
- TLS 1.2 minimum
- Strong cipher suites
- HSTS with `includeSubDomains`

No need to manually tune.

---

## Logging Sensitive Data

Never log:
- Passwords (even hashed)
- JWT tokens
- API keys
- Full credit card numbers (no payment for MVP anyway)
- Voice transcripts in production logs (they may contain PII)

Do log (with care):
- User IDs (numeric, not emails)
- Endpoint paths
- Status codes
- Latencies
- Error types (not full stack traces with data)

Use structured logging (JSON) so it's easy to filter sensitive fields.

---

## Dependency Security

### Backend
- Run `pip-audit` weekly to check for CVEs
- Pin dependencies (`requirements.txt` with exact versions)
- Update minor versions monthly
- Update major versions only with manual review

### Frontend
- Run `npm audit` weekly
- Use `package-lock.json` for reproducibility
- Same update cadence

### Docker Images
- Use specific image tags (not `latest`)
- Rebuild with new base images monthly
- Scan with `trivy` (optional, can defer)

---

## Incident Response

### If User Reports Compromised Account
1. Force logout (invalidate all refresh tokens for that user_id)
2. Force password reset email
3. Review audit_log for suspicious activity
4. Suspend account if necessary

### If Mass Cost Spike Detected
1. Disable debate creation globally (env flag)
2. Investigate via logs
3. Identify abusing user(s) or compromised key
4. Rotate API keys if necessary
5. Re-enable when safe

### If Database Compromise Suspected
1. Immediately rotate all secrets
2. Force logout all users (rotate JWT_SECRET)
3. Force password reset for all users
4. Audit logs to determine scope
5. Notify users (legal obligation depending on data exposed)

---

## Things We Are NOT Doing for MVP

For honesty, here's what we explicitly skip:

- 2FA / MFA — Defer to v2
- OAuth (Google/Facebook login) — Defer to v2
- WebAuthn / passkeys — Defer to v3
- Comprehensive audit logging UI — DB queries only for now
- Automated penetration testing — Manual review only
- Bug bounty program — Defer to post-launch
- Compliance certifications (SOC2, ISO 27001) — Not needed at this scale
- Web Application Firewall (WAF) — Cloudflare WAF can be added later if needed
- DDoS protection at scale — Cloudflare can be added later

These are all important eventually. Just not for getting the MVP shipped.
