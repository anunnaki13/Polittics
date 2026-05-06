# Spec 07 — Share to Social Media

## Purpose

Make sharing a clip to TikTok, Instagram, WhatsApp, or other platforms friction-free. The sharing flow is critical for organic growth — every shared clip is potential new user acquisition.

## Target Platforms (Priority Order)

1. **TikTok** — Primary target. Indonesian Gen Z audience.
2. **Instagram (Reels + Stories)** — Secondary, large user base.
3. **WhatsApp** — For 1-on-1 sharing with friends.
4. **Twitter/X** — Power users, political discourse.
5. **Generic copy link** — Catch-all.

## Sharing Mechanisms

### Option A: Native Share Sheet (Mobile)

On mobile browsers (Chrome Android, Safari iOS), use Web Share API:

```typescript
async function shareClip(debate: Debate) {
  const shareMeta = await api.clips.getShareMeta(debate.id);
  
  // Try sharing with file attachment first (best experience)
  try {
    const response = await fetch(shareMeta.clip_url);
    const blob = await response.blob();
    const file = new File([blob], 'arena-politika-debat.mp4', { type: 'video/mp4' });
    
    if (navigator.canShare?.({ files: [file] })) {
      await navigator.share({
        title: shareMeta.title,
        text: shareMeta.description,
        files: [file],
      });
      return;
    }
  } catch (err) {
    // Fall through to URL-only share
  }
  
  // Fallback: share URL only
  if (navigator.share) {
    await navigator.share({
      title: shareMeta.title,
      text: shareMeta.description,
      url: shareMeta.clip_url,
    });
    return;
  }
  
  // Desktop fallback: show modal with platform buttons
  openShareModal(shareMeta);
}
```

### Option B: Platform-Specific Buttons (Desktop & Fallback)

Show 5 buttons for desktop or when native share isn't available:

```
┌──────────────────────────────────────┐
│  Bagikan Klip                        │
│                                      │
│  [📱 TikTok]  [📷 Instagram]         │
│  [💬 WhatsApp] [🐦 Twitter/X]        │
│  [🔗 Salin Link]                     │
│                                      │
│  [⬇ Unduh Klip MP4]                  │
└──────────────────────────────────────┘
```

#### TikTok
TikTok doesn't have a direct web share intent. Strategy:
1. Auto-download the MP4 to user's device
2. Show modal with instructions
3. Pre-filled caption shown for user to copy

```typescript
function shareToTikTok(shareMeta: ShareMeta) {
  // Trigger download
  downloadFile(shareMeta.clip_url, 'arena-politika.mp4');
  
  // Show instructions modal
  showModal({
    title: 'Bagikan ke TikTok',
    instructions: [
      '1. Klip MP4 telah diunduh ke perangkat Anda',
      '2. Buka aplikasi TikTok',
      '3. Tap tombol [+] untuk upload video',
      '4. Pilih video yang baru diunduh',
      '5. Salin caption di bawah:',
    ],
    captionToCopy: shareMeta.share_text_tiktok,
  });
  
  // Track share intent
  api.clips.trackShare(shareMeta.debate_id, 'tiktok');
}
```

#### Instagram
Same pattern as TikTok. Instagram has `instagram://share` URL scheme but unreliable across platforms.

```typescript
function shareToInstagram(shareMeta: ShareMeta) {
  downloadFile(shareMeta.clip_url, 'arena-politika.mp4');
  showModal({
    title: 'Bagikan ke Instagram',
    instructions: [
      '1. Klip telah diunduh',
      '2. Buka Instagram',
      '3. Tap [+] → Reel atau Story',
      '4. Upload video yang baru diunduh',
      '5. Salin caption:',
    ],
    captionToCopy: shareMeta.share_text_instagram,
  });
  api.clips.trackShare(shareMeta.debate_id, 'instagram');
}
```

#### WhatsApp
WhatsApp has a robust web intent:

```typescript
function shareToWhatsApp(shareMeta: ShareMeta) {
  const text = `${shareMeta.share_text_whatsapp}\n\n${shareMeta.public_url}`;
  const url = `https://wa.me/?text=${encodeURIComponent(text)}`;
  window.open(url, '_blank');
  api.clips.trackShare(shareMeta.debate_id, 'whatsapp');
}
```

#### Twitter/X
Twitter share intent supports text + URL:

```typescript
function shareToTwitter(shareMeta: ShareMeta) {
  const text = `${shareMeta.share_text_twitter}\n\n${shareMeta.public_url}`;
  const url = `https://twitter.com/intent/tweet?text=${encodeURIComponent(text)}`;
  window.open(url, '_blank');
  api.clips.trackShare(shareMeta.debate_id, 'twitter');
}
```

#### Copy Link
```typescript
async function copyClipLink(shareMeta: ShareMeta) {
  await navigator.clipboard.writeText(shareMeta.public_url);
  showToast('Link disalin ke clipboard');
  api.clips.trackShare(shareMeta.debate_id, 'copy_link');
}
```

#### Direct Download
```typescript
function downloadClip(shareMeta: ShareMeta) {
  downloadFile(shareMeta.clip_url, 'arena-politika.mp4');
  api.clips.trackShare(shareMeta.debate_id, 'download');
}

function downloadFile(url: string, filename: string) {
  const link = document.createElement('a');
  link.href = url;
  link.download = filename;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}
```

## Share Text Templates

Pre-generated server-side (in `GET /api/clips/{id}/share-meta`). Templates per platform:

### TikTok / Instagram (short, hashtag-heavy)
```
Skor debat politikku: {score}/100 🔥
Topik: {topic_motion_short}
{result_emoji} {result}

Coba juga di arenapolitika.id

#ArenaPolitika #DebatPolitik #IndonesiaDebat #AIChallenge #PolitikGenZ
```

### WhatsApp (conversational)
```
Aku barusan main Arena Politika!
Topik: "{topic_motion}"
Skor aku: {score}/100 ({result_indonesian})

Coba kamu juga: {public_url}
```

### Twitter/X (concise, punchy, under 280 chars)
```
Skor {score}/100 di Arena Politika 🎯
"{topic_motion_short}" — {result_indonesian}

Coba debat AI kamu di {public_url}
```

### Generic (used for native share API)
```
Saya baru saja debat tentang {topic_motion_short} dan mendapat skor {score}/100. Coba di Arena Politika!
```

## Implementation Notes

### Result Emoji Mapping
- MENANG → 🏆
- KALAH → 💪 (defiant, not loser)
- SERI → ⚖️

### Topic Motion Truncation
For social platforms, limit topic motion to 60 chars + ellipsis. Keep meaning intact:
- Original: "Subsidi BBM harus dicabut total demi efisiensi anggaran negara"
- Short: "Subsidi BBM harus dicabut total demi efisiensi"

Truncation algorithm: cut at last word boundary before 60 chars, append "...".

### Hashtag Strategy
Use 3-5 hashtags. Mix of:
- Brand: `#ArenaPolitika` (always)
- Category: `#DebatPolitik` (always)
- Generic Indonesian: `#IndonesiaDebat`
- Trending (rotate): `#AIChallenge`, `#GenZ`, `#PolitikGenZ`

Don't overuse hashtags. TikTok/Instagram algorithms penalize spam.

## Backend Endpoint

```python
# app/clips/router.py
@router.get("/{debate_id}/share-meta", response_model=ShareMeta)
async def get_share_meta(
    debate_id: UUID,
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    debate = await get_debate_for_user(db, debate_id, user.id)
    if not debate.clip_url:
        raise NotFoundError("Klip belum tersedia")
    if debate.status != "complete":
        raise NotFoundError("Debat belum selesai")
    
    public_url = f"{settings.APP_BASE_URL}/c/{debate_id}"
    score = int(debate.scores.total)
    motion_short = truncate_motion(debate.topic.motion, 60)
    result_id = translate_result(debate.scores.result)  # MENANG -> Menang
    result_emoji = RESULT_EMOJI[debate.scores.result]
    
    return ShareMeta(
        debate_id=debate.id,
        clip_url=debate.clip_url,
        thumbnail_url=debate.clip_url.replace(".mp4", "-thumb.jpg"),
        public_url=public_url,
        title=f"Skor {score}/100 di Arena Politika!",
        description=f"Saya baru saja debat tentang {motion_short} dan mendapat skor {score}/100",
        share_text_tiktok=build_tiktok_text(score, motion_short, result_id, result_emoji),
        share_text_instagram=build_instagram_text(score, motion_short, result_id, result_emoji),
        share_text_whatsapp=build_whatsapp_text(score, debate.topic.motion, result_id, public_url),
        share_text_twitter=build_twitter_text(score, motion_short, result_id, public_url),
    )


@router.post("/{debate_id}/track-share")
async def track_share(
    debate_id: UUID,
    data: TrackShareRequest,  # {platform: str}
    user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """Track which platform user clicked. Used for analytics only."""
    await audit_service.log_event(
        db,
        user_id=user.id,
        event_type="clip_shared",
        metadata={"debate_id": str(debate_id), "platform": data.platform},
    )
    return {"ok": True}
```

## Open Graph & Twitter Cards (Public Preview Page)

When the public clip URL `/c/{debate_id}` is shared (e.g., paste in Twitter, WhatsApp), the link preview should show:
- Thumbnail (first frame of video)
- Title
- Description
- Video preview if platform supports

Implementation: serve a special HTML page at `/c/{debate_id}` (public, no auth) that has:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Arena Politika - Debat Skor {score}</title>
  
  <!-- Open Graph -->
  <meta property="og:type" content="video.other">
  <meta property="og:title" content="Skor {score}/100 di Arena Politika!">
  <meta property="og:description" content="Debat tentang {topic_short}. Coba juga di arenapolitika.id">
  <meta property="og:image" content="{thumbnail_url}">
  <meta property="og:image:width" content="1080">
  <meta property="og:image:height" content="1920">
  <meta property="og:video" content="{clip_url}">
  <meta property="og:video:secure_url" content="{clip_url}">
  <meta property="og:video:type" content="video/mp4">
  <meta property="og:video:width" content="1080">
  <meta property="og:video:height" content="1920">
  <meta property="og:url" content="{public_url}">
  <meta property="og:site_name" content="Arena Politika">
  
  <!-- Twitter Card -->
  <meta name="twitter:card" content="player">
  <meta name="twitter:title" content="Skor {score}/100 di Arena Politika!">
  <meta name="twitter:description" content="Debat AI Bahasa Indonesia">
  <meta name="twitter:image" content="{thumbnail_url}">
  <meta name="twitter:player" content="{public_url}/embed">
  <meta name="twitter:player:width" content="540">
  <meta name="twitter:player:height" content="960">
  
  <!-- No-index for search engines -->
  <meta name="robots" content="noindex, nofollow">
  
  <!-- Auto-redirect logged-in users to detail page -->
  <script>
    if (typeof window !== 'undefined' && document.cookie.includes('refresh_token')) {
      window.location.href = '/riwayat/{debate_id}';
    }
  </script>
</head>
<body>
  <main>
    <h1>Klip Debat Arena Politika</h1>
    <video controls autoplay muted poster="{thumbnail_url}">
      <source src="{clip_url}" type="video/mp4">
    </video>
    <p>Coba debat AI sendiri di <a href="https://arenapolitika.id">arenapolitika.id</a></p>
  </main>
</body>
</html>
```

This page is generated server-side by FastAPI returning HTML response. Crawlers (Twitter, WhatsApp, Discord, Slack) read OG tags and show rich preview.

### Thumbnail Generation
Generate thumbnail at clip generation time (1 frame extracted at second 2):

```bash
ffmpeg -i clip.mp4 -ss 2 -vframes 1 -vf "scale=1080:1920" thumb.jpg
```

Stored alongside clip in MinIO as `clips/{debate_id}-thumb.jpg`.

## Privacy Considerations

### Public Clip Access
The clip URL is technically public — anyone with the URL can view it. This is intentional (so social platforms can preview it).

But:
- URLs contain UUIDs (not enumerable, ~10^38 possibilities)
- `robots.txt` blocks `/clips/*` and `/c/*` from search indexing
- User can request clip deletion at any time

### Default-Private Behavior (v2)
For MVP, clips are technically accessible to anyone with URL. In v2, add explicit toggle: "Bagikan klip ini secara publik?" before share buttons enable. Generate clip privately by default.

For MVP, this risk is acceptable because:
- UUID-based URLs are not enumerable
- Users explicitly click share to expose URL
- No PII in clips beyond user's voice + initials

### Voice Privacy
The clip contains the user's voice. By sharing, user is consenting to public exposure of their voice on social media. This is normal and expected.

## Frontend Component

```typescript
// components/debate/ShareButtons.tsx
interface ShareButtonsProps {
  debate: Debate;
}

export function ShareButtons({ debate }: ShareButtonsProps) {
  const { data: shareMeta, isLoading } = useQuery({
    queryKey: ['share-meta', debate.id],
    queryFn: () => api.clips.getShareMeta(debate.id),
    enabled: !!debate.clip_url,
  });
  
  if (!debate.clip_url) {
    return <div className="text-muted">Klip belum tersedia</div>;
  }
  
  if (isLoading || !shareMeta) {
    return <SkeletonButtons count={5} />;
  }
  
  // Mobile: show single native share button + download
  if (isMobile()) {
    return (
      <div className="flex flex-col gap-2">
        <Button onClick={() => shareClip(debate)} variant="primary" size="lg">
          📤 Bagikan
        </Button>
        <Button onClick={() => downloadClip(shareMeta)} variant="outline">
          ⬇ Unduh Klip
        </Button>
      </div>
    );
  }
  
  // Desktop: show platform grid
  return (
    <div className="flex flex-col gap-3">
      <h3 className="text-lg font-semibold">Bagikan Klip</h3>
      <div className="grid grid-cols-2 sm:grid-cols-5 gap-2">
        <PlatformButton platform="tiktok" onClick={() => shareToTikTok(shareMeta)} />
        <PlatformButton platform="instagram" onClick={() => shareToInstagram(shareMeta)} />
        <PlatformButton platform="whatsapp" onClick={() => shareToWhatsApp(shareMeta)} />
        <PlatformButton platform="twitter" onClick={() => shareToTwitter(shareMeta)} />
        <PlatformButton platform="copy" onClick={() => copyClipLink(shareMeta)} />
      </div>
      <Button onClick={() => downloadClip(shareMeta)} variant="outline">
        ⬇ Unduh Klip MP4
      </Button>
    </div>
  );
}
```

## Tracking (Light Analytics)

Track share button clicks for analytics (which platforms users prefer):

The `track_share` endpoint logs to `audit_log` table with:
- user_id
- event_type: `clip_shared`
- metadata: `{debate_id, platform}`

Used for weekly analysis: which platforms drive most shares? Optimize accordingly.

This is server-side analytics. No third-party trackers (no GA, no Facebook Pixel) for MVP.

## Edge Cases

### Clip URL Expired or Deleted
If user accesses old shared link after clip is deleted (account deletion, manual removal):
- Show 404 page: "Klip ini sudah tidak tersedia"
- Suggest creating own debate

### Mobile Browser Incompatibility
Some older mobile browsers don't support Web Share API or file sharing.
- Fallback to platform-specific buttons (same as desktop)
- Always provide download option as ultimate fallback

### Large File on Slow Connection
3 MB clip on 3G can take 30+ seconds to download for sharing.
- Show progress indicator during download
- Allow cancel
- Suggest WiFi if on cellular

### TikTok / Instagram App Not Installed
Modal instructions assume app is installed. For desktop users:
- Add note: "Pastikan aplikasi TikTok/Instagram terinstal di HP Anda"
- Suggest sending link via WhatsApp to phone

## Acceptance Criteria

- [ ] Native share sheet works on Android Chrome and iOS Safari
- [ ] Native share includes video file (not just URL) when supported
- [ ] All 5 platform buttons work on desktop (Chrome, Firefox, Safari)
- [ ] TikTok/Instagram flow downloads MP4 + shows instructions
- [ ] WhatsApp opens with pre-filled text in new tab
- [ ] Twitter opens compose with pre-filled text + URL
- [ ] Copy link copies and shows toast
- [ ] Public preview page (`/c/{id}`) renders OG tags correctly
- [ ] Pasting clip URL in WhatsApp shows rich preview
- [ ] Pasting clip URL in Twitter shows video player card
- [ ] Thumbnail generated and accessible
- [ ] Share tracking logged to audit_log
- [ ] All UI text in Bahasa Indonesia
- [ ] Tests pass for share text generation logic

## Out of Scope for MVP

- Direct upload to TikTok/Instagram via API (requires partnership and OAuth)
- Custom share text editing by user before posting
- Branded link shortener (use raw URL)
- Share to LinkedIn, Reddit, Discord (defer based on alpha feedback)
- Embedded player widget for blogs/websites
- Schedule share for later
- Group shares (share to multiple platforms at once)
- Watermark customization per user
- Share analytics dashboard for users
