# PRD: Motion Reels Autopilot

Personal tool to turn audio files into motion graphics Reels/Shorts and auto-post them to Instagram and YouTube on a fixed schedule.

- **Owner / only user:** Dharm
- **Status:** Draft v1
- **Hosting:** Vercel (web) + Railway (API + worker) + Supabase (Postgres + Storage)

---

## 1. Problem

Roz 2 motion graphics videos (morning + evening) banana, edit karna, title/caption/tags likhna aur dono platforms par manually upload karna time-consuming hai. Goal hai ek aisa pipeline jahan sirf audio files upload karni hain aur baaki sab automatic ho.

## 2. Goals

1. Ek saath N audio files (typically 6) upload karo, system unhe queue me daal de.
2. AI agent har audio ke liye vertical motion graphics video (1080x1920) banaye, motion-skills ki madad se.
3. Agent hi **title, caption, hashtags/tags, description** likhe.
4. Jaise-jaise video ready ho, wo agle free slot (morning / evening) par schedule ho. 6 videos = 3 din.
5. Instagram Reels aur YouTube Shorts dono par auto-post.
6. Jab batch ke sab videos schedule ho jayein, **email notification** aaye.
7. Scheduled video ka file storage se **7 din ke andar delete** ho jaye.

## 3. Non-goals (v1)

- Multi-user / public SaaS, billing, teams.
- TikTok, Facebook, LinkedIn, X posting.
- Manual timeline editor.
- Analytics dashboard (baad me).

## 4. Assumptions (confirm / change karo)

| # | Assumption | Default |
|---|------------|---------|
| A1 | Posting slots (IST) | 09:00 aur 19:00 |
| A2 | Videos per day | 2 |
| A3 | Video length | Audio length ke barabar, target 15-60 sec |
| A4 | Approval step | Off (fully automatic), toggle available |
| A5 | Retention | Scheduled hone ke 7 din baad delete, par publish hone ke baad hi |
| A6 | Audio | Tumhari apni / rights-cleared voiceover ya music |
| A7 | Language of captions | Audio ki language ke hisaab se (Hindi / English / Hinglish) |

## 5. Functional requirements

### FR-1 Auth
- Dashboard single-user hai par public internet par hai, isliye login zaroori (Supabase Auth, sirf ek allowed email).

### FR-2 Connect accounts (OAuth)
- "Connect YouTube" (Google OAuth, scope: `youtube.upload`) aur "Connect Instagram" (Meta OAuth, content publishing).
- Tokens encrypted store hon, expiry track ho, near-expiry par auto-refresh.
- Dashboard par connection status (connected / expiring / broken) dikhe.

### FR-3 Batch upload
- Multiple audio files (mp3, wav, m4a) select karke upload.
- Browser se **direct Supabase Storage** par upload (signed upload URL), Vercel ke through nahi (body size limit).
- Upload ke baad ek `batch` aur har file ke liye ek `job` ban jaye (status `queued`).

### FR-4 Queue and processing
- Worker jobs ko ek ek karke (FIFO, concurrency = 1) uthaye.
- Pipeline per job: transcribe, plan, generate video, verify, generate metadata, upload final mp4.
- Retries: max 2, phir `failed` aur email alert. Failed job batch ko block na kare, baaki jobs chalte rahein.

### FR-5 AI video generation
- Agent ko transcript + word timestamps + audio + brand/style config milte hain.
- Agent motion-skills use karke Remotion project likhta hai aur render karta hai (details section 8).
- Output: mp4, 1080x1920, 30 fps, H.264 + AAC, duration audio se match.
- Automatic verification (ffprobe + frame sampling) fail ho to retry.

### FR-6 Metadata generation
- Har video ke liye JSON: YouTube title, YouTube description, YouTube tags, Instagram caption, hashtags.
- Platform limits enforce hon (section 10).

### FR-7 Scheduling
- Video `ready` hote hi next free slot assign ho (section 9).
- Dashboard me calendar/list view, aur slot manually change / skip karne ka option.

### FR-8 Publishing
- Scheduler (Railway worker) har minute due posts check kare.
- YouTube: Data API v3 `videos.insert` (resumable upload), Shorts-compatible vertical video + `#Shorts`.
- Instagram: container create, status poll, `media_publish`.
- Har platform ka result alag track ho (ek fail ho to dusra chal sakta hai).

### FR-9 Notifications (email)
- **Batch scheduled:** jab batch ke saare jobs `scheduled` (ya `failed`) ho jayein, ek summary email: title + date/time + platform per video.
- **Failure:** job fail ya publish fail par email.
- **Token expiring / broken:** reconnect karne ka email.

### FR-10 Cleanup
- Daily cleanup job: jin videos ka `delete_after` nikal gaya aur publish ho chuka, unka mp4 aur source audio storage se delete.
- DB record rahe (title, links, status), sirf files delete hon.

### FR-11 Dashboard
- Batches, per-job status/progress, logs, preview player, retry button, schedule view, settings (slots, style, approval toggle).

## 6. Architecture

```
                 +-------------------+
 Browser  -----> |  Vercel: Next.js  |  (UI, Supabase Auth, signed uploads)
                 +---------+---------+
                           |  REST + Supabase Realtime
                 +---------v---------+
                 | Railway: FastAPI  |  (batches, OAuth callbacks, retry, settings)
                 +---------+---------+
                           |
        +------------------v------------------+
        |        Supabase (Postgres)          |  jobs, schedule, tokens, events
        |        Supabase Storage             |  audio/, video/
        +------------------+------------------+
                           ^
                 +---------+---------+
                 | Railway: Worker   |  queue consumer, AI agent, Remotion render,
                 | (same image)      |  scheduler, publisher, cleanup, email
                 +-------------------+
```

**Kyun Railway worker?** Vercel serverless par long render, ffmpeg/Chromium aur persistent cron-style scheduler practical nahi hain. Worker ek long-running process hai jo ye sab handle karta hai, aur Vercel cron ki limits ka issue bhi khatam.

## 7. Tech stack

| Layer | Choice |
|-------|--------|
| Web | Next.js (App Router), TypeScript, Tailwind, Supabase JS client |
| API | Python 3.12, FastAPI, Pydantic |
| Worker | Python (same repo/image), APScheduler for time-based jobs |
| DB | Supabase Postgres, migrations via Supabase CLI or Alembic |
| Queue | Postgres table + `FOR UPDATE SKIP LOCKED` (Redis ki zaroorat nahi) |
| Storage | Supabase Storage (private buckets, signed URLs) |
| Video | Remotion (Node 20) + Chromium + ffmpeg inside the Railway image |
| Transcription | faster Whisper API (word-level timestamps) |
| LLM / Agent | Pluggable runner: Claude Code headless ya OpenCode CLI (config se select) |
| Email | Resend |
| Secrets | Railway / Vercel env vars, OAuth tokens DB me encrypted (Fernet) |

> Remotion Node-based hai, isliye Python worker use subprocess (`npx remotion render`) se chalata hai.

## 8. AI agent design

### 8.1 Inputs per job
- `audio.mp3`, `transcript.json` (words + timestamps), audio duration
- `style.json` (colors, fonts, logo, caption style, safe areas)
- Prompt template (section 8.4)

### 8.2 Skills (iart-ai/motion-skills)
Repo ke README ke mutabiq ye ek hub hai jisme alag-alag installable packs hain. Shorts ke liye starting set:

| Pack | Kyun |
|------|------|
| `tiktok-video-skills` | Reels / Shorts / TikTok vertical format |
| `kinetic-typography-skills` | Word-synced animated text |
| `b-roll-skills` | Bole gaye words se time kiye hue motion b-roll |
| `explainer-video-skills` | Educational / explainer style clips |
| `motion-design-skills` (optional) | Logo/brand motion, easing, polish |
| `data-animation-skills` (optional) | Agar audio me numbers/stats ho |

Install (Docker build ke time, agent workspace ke andar):

```bash
npx skills add iart-ai/tiktok-video-skills
npx skills add iart-ai/kinetic-typography-skills
npx skills add iart-ai/b-roll-skills
npx skills add iart-ai/explainer-video-skills
```

README ke hisaab se skills Claude Code, Cursor, Codex jaise agents me auto-discover hoti hain. OpenCode use karo to check karo ki wo skills folder kahan padhta hai, zarurat ho to skill folders wahan copy karo. Exact skill names har pack ke repo me check kar lena, mainne sirf hub README dekha hai.

### 8.3 Agent workspace
```
agent-workspace/
  .claude/skills/ (ya OpenCode ka equivalent)   # installed packs
  remotion-base/                                 # pre-configured Remotion project
  jobs/<job_id>/{audio.mp3, transcript.json, style.json, out/}
```
Har job ke liye fresh copy of `remotion-base`, taaki ek job doosre ko kharab na kare.

### 8.4 Agent task (prompt outline)
1. Transcript padho, content ka angle/hook decide karo.
2. Storyboard: scenes with start/end timestamps (hook pehle 1-2 sec me).
3. Remotion composition likho (kinetic captions + motion graphics), skills follow karte hue.
4. Render `out/video.mp4` (1080x1920, 30fps).
5. Self-verify: ek frame freeze karo, contact sheet banao, text overflow / safe-area check karo, fix karo.
6. `out/metadata.json` likho (section 10 ka schema).

### 8.5 Automatic verification (worker side, agent ke baad)
- `ffprobe`: resolution 1080x1920, codec H.264/AAC, audio track present.
- Video duration = audio duration +/- 0.5s.
- File size < platform limits, duration <= 60s (configurable).
- Fail hua to job retry (agent ko error message ke saath).

### 8.6 Guardrails
- Agent **sandboxed container** me chalta hai, kyunki wo code likhta aur execute karta hai. Headless mode me permissions auto-approve karni padti hain, isliye us container me production secrets (OAuth tokens, service keys) **mount na karo**. Agent ko sirf job directory aur LLM key milegi.
- Hard timeout per job (default 20 min), max 2 retries.
- Network access agent step me minimum (sirf LLM API aur npm/fonts jo zaroori hon).
- Agent ko publishing ya DB access nahi milta; wo sirf files banata hai.

## 9. Scheduling rules

- Slots: `slot_times = ["09:00", "19:00"]`, timezone `Asia/Kolkata`, configurable.
- Jab job `ready` ho: **next free slot** dhoondo jo `now + buffer (30 min)` ke baad ho aur jisme koi dusra video na ho.
- Videos completion order me slots lete hain. Isse 6 videos ke liye 3 din ki queue banti hai.
- Agar batch ke beech me naya batch aaye to slots continue hote hain (koi slot double-book nahi hoga, unique constraint on `slot_at`).
- Failed job ka slot khali reh jata hai aur agla ready video use fill kar sakta hai (optional compaction setting).
- Missed slot (worker down tha): due post jaise hi worker wapas aaye, turant publish ho, jab tak delay `late_tolerance` (default 6 hrs) ke andar ho, warna `needs_attention` + email.

## 10. Metadata spec

AI se structured JSON:

```json
{
  "youtube": {
    "title": "max 100 chars, hook-first",
    "description": "max 5000 chars, #Shorts included",
    "tags": ["..."]
  },
  "instagram": {
    "caption": "max 2200 chars",
    "hashtags": ["max 30, ideally 5-10"]
  },
  "hook": "one line summary of the first 2 seconds",
  "language": "hi | en | hinglish"
}
```

Validation (code me, LLM par bharosa nahi): title <= 100, tags combined <= 500 chars, caption <= 2200, hashtags <= 30. Limit cross ho to auto-trim.

## 11. Data model (Postgres)

```sql
batches(id, created_at, status, total_jobs, notified_at)

jobs(
  id, batch_id, status,            -- see flow.md state machine
  audio_path, video_path,
  transcript jsonb, metadata jsonb,
  attempts int, error text,
  created_at, started_at, finished_at
)

schedule(
  id, job_id unique, slot_at timestamptz unique,
  scheduled_at timestamptz,        -- jab schedule hua
  delete_after timestamptz,        -- scheduled_at + 7 days
  status                           -- pending | publishing | published | failed
)

publications(
  id, job_id, platform,            -- youtube | instagram
  status, external_id, external_url,
  attempts, error, published_at
)

social_accounts(
  id, platform, account_label,
  access_token_enc, refresh_token_enc,
  expires_at, status
)

settings(key, value jsonb)         -- slots, tz, style, approval toggle
events(id, job_id, level, message, created_at)   -- logs for dashboard
```

## 12. Storage and retention

- Buckets: `audio` (input), `video` (output), dono private.
- Instagram ko **public URL** chahiye video fetch karne ke liye, isliye publish ke time par signed URL banao (lambi expiry, jaise 1-2 hrs) aur container `FINISHED` hone tak valid rakho.
- Retention rule: `delete_after = scheduled_at + 7 days`. Cleanup tabhi delete kare jab dono platform publications `published` hon. Agar kuch pending/failed hai to delete defer ho.
- Delete hone par `jobs.video_path = null`, record bacha rahe.

## 13. Integrations: constraints jo dhyan me rakhne hain

### YouTube
- Google Cloud project, YouTube Data API v3 enable, OAuth consent screen.
- Consent screen **Testing** me rakhoge to refresh token ~7 din me expire hota hai, isliye **In production** me push karo (personal use me unverified-app warning aayegi, chalega).
- Unverified API projects se upload hue videos **private lock** ho sakte hain jab tak API compliance audit na ho. Pehle ek test upload karke verify karo.
- Default quota 10,000 units/day, upload ~1,600 units, 2 videos/day ke liye kaafi.
- Shorts: vertical, 3 min se kam (Shorts ki limit change hoti rehti hai, current docs check karo).

### Instagram
- Professional (Business/Creator) account chahiye.
- Meta app **Development mode** me apne account ke liye chalta hai, App Review ki zaroorat nahi.
- Do possible login paths hain: Instagram API with Instagram Login ya Facebook Login (Page se linked). Implementation se pehle current Meta docs me dekho ki tumhare account ke liye kaun sa simple hai.
- Publishing flow: `POST /media` (REELS, `video_url`) -> status poll -> `POST /media_publish`.
- Native scheduling API me nahi hai, isliye hamara scheduler publish karta hai.
- Long-lived token ~60 din; expiry se pehle refresh job zaroori.

### Email
- Resend (ya SMTP). Sender domain/email verify karna padega.

## 14. Security

- Dashboard login required, Supabase RLS on, service-role key sirf Railway par.
- OAuth tokens DB me encrypted, encryption key env var me.
- Railway API endpoints par auth (Supabase JWT verify) + OAuth `state` parameter check.
- Signed URLs short-lived (sirf publish window ke liye).
- Agent container isolation (section 8.6).
- Secrets git me kabhi nahi.

## 15. Non-functional

| Area | Target |
|------|--------|
| Reliability | Publish miss < 1% (retries + late tolerance) |
| Throughput | 1 video at a time, 6 videos batch me pura queue kuch ghanton me nikal jaye |
| Observability | Har state transition `events` me log, dashboard par dikhe |
| Cost | Per video: faster Whisper + LLM tokens + Railway compute. Ek test batch ke baad actual cost measure karo |
| Resources | Remotion render ke liye Railway service ko kam se kam kuch GB RAM chahiye, shuru me test karke size choose karo |

## 16. Risks and open questions

| Risk | Mitigation |
|------|------------|
| AI-generated video quality inconsistent | Verification loop, optional approval toggle, style.json se consistency, shuru ke kuch batches manually review |
| Agent run slow / costly | Timeout, retries cap, reusable `remotion-base`, cost logging per job |
| YouTube audit / private lock | Early test upload, audit form apply |
| Meta/Google token expiry | Refresh cron + email alerts |
| Repetitive templated content par platform policy (jaise YouTube ki inauthentic/mass-produced content policy) | Videos me genuine variation aur apna creative input rakho, policies periodically check karo |
| Audio copyright claims | Sirf apni ya licensed audio use karo |
| Remotion licensing | Remotion ki license terms apne use-case ke liye check karo |
| Headless agent runs code | Sandbox container, no secrets mounted |

**Open questions:**
1. Exact morning/evening times (default 09:00 / 19:00 IST)?
2. Approval step on ya off?
3. Audio kis type ki hai (voiceover, music, dono)? Isse style strategy badlegi.
4. Kaun sa agent runner: Claude Code headless ya OpenCode?
5. Content niche/brand style (fonts, colors, logo) kya hai?

## 17. Milestones

| M | Deliverable |
|---|-------------|
| M0 | Repo + Railway image (Python + Node + Chromium + ffmpeg), Supabase schema, one manual Remotion render |
| M1 | Audio upload, queue, faster Whisper transcript, agent generates 1 video end-to-end |
| M2 | Metadata generation + verification + retries |
| M3 | YouTube OAuth + upload (test video) |
| M4 | Instagram OAuth + Reels publish |
| M5 | Scheduler + slot allocation + batch-complete email |
| M6 | Cleanup job + token refresh + failure alerts |
| M7 | Dashboard polish, approval toggle, real batch of 6 |

## 18. Repo structure

```
motion-reels-autopilot/
  apps/
    web/                    # Next.js on Vercel
    api/                    # FastAPI + worker (Railway)
      app/
        main.py
        routes/             # batches, oauth, jobs, settings
        worker/
          queue.py
          pipeline.py
          agent_runner.py
          verify.py
          scheduler.py
          publishers/{youtube.py, instagram.py}
          cleanup.py
          emailer.py
      Dockerfile
  agent-workspace/
    remotion-base/
    skills/                 # installed motion-skills packs
  supabase/migrations/
  docs/{PRD.md, flow.md}
```

## 19. Environment variables

```
# Shared / Supabase
SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY, DATABASE_URL

# Railway (API + worker)
LLM_API_KEY / AGENT_RUNNER  # claude-code | opencode
TOKEN_ENCRYPTION_KEY
GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET, GOOGLE_REDIRECT_URI
META_APP_ID, META_APP_SECRET, META_REDIRECT_URI
RESEND_API_KEY, NOTIFY_EMAIL
TIMEZONE=Asia/Kolkata

# Vercel (web)
NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, NEXT_PUBLIC_API_URL
```
