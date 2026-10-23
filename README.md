# Motion Reels Autopilot

Audio files in, scheduled YouTube Shorts and Instagram Reels out. Built for one
operator: upload a batch, review the rendered vertical video, approve it, and
let the worker publish at 09:00 / 19:00 IST.

## Stack

| Layer | Choice |
|---|---|
| Dashboard | Next.js 16 + shadcn/ui + Tailwind v4 |
| API | FastAPI (Python 3.12) |
| Worker | Postgres queue (`FOR UPDATE SKIP LOCKED`) + APScheduler |
| Transcription | `faster-whisper==1.2.1` + `av==15.1.0`, `small` + `int8`, dedicated process pool |
| Video | Remotion fallback renderer (first path), optional OpenCode/Claude agent |
| Storage | Supabase Storage (`audio`, `video` buckets) |
| Database | Supabase Postgres (direct/session connection preferred) |
| Email | Resend |
| Publishing | YouTube Shorts first, then Instagram Reels |
| Retention | Delete audio + video 7 days after both platforms publish |

## Repository layout

```
apps/web                 Next.js dashboard (/dashboard)
apps/api                 FastAPI + worker
agent-workspace          Remotion base, agent skills, per-job workspaces
supabase/migrations      Tables, partial unique slot index, RLS, buckets
plan.md                  Approved implementation plan + locked corrections
PRD.md / flow.md         Product requirements and state machine
```

## Local setup

### 1. Prerequisites

- Python **3.12**
- Node.js **22+** and pnpm
- ffmpeg / ffprobe on `PATH`
- A Supabase project (or local Postgres plus Supabase Storage)

### 2. Environment

```bash
cp .env.example .env
cp apps/web/.env.example apps/web/.env.local
```

Fill at least:

- `DATABASE_URL` — direct/session connection on port `5432`.
  If you use the transaction pooler on `6543`, keep
  `DB_STATEMENT_CACHE_SIZE=0` and avoid session-level features.
- `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY`
- `SUPABASE_JWT_SECRET` **or** asymmetric signing keys + `SUPABASE_URL` for
  JWKS. Check Supabase → Settings → API → JWT signing keys.
- `TOKEN_ENCRYPTION_KEY`:
  ```bash
  python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
  ```
- `RESEND_API_KEY`, `NOTIFY_EMAIL` (optional locally)
- OAuth credentials when you are ready to publish.

`AUTH_DISABLED=true` is allowed only in development. With
`ENVIRONMENT=production`, the API refuses to boot unless auth is configured.

### 3. Python 3.12 venv

```bash
cd apps/api
py -3.12 -m venv .venv
# Windows
.venv\Scripts\activate
# macOS/Linux
# source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 4. Database and storage

Apply `supabase/migrations/0001_init.sql` with the Supabase CLI or `psql`:

```bash
supabase db push
# or
psql "$DATABASE_URL" -f supabase/migrations/0001_init.sql
```

The migration creates:

- tables (`batches`, `jobs`, `schedule`, `publications`, `social_accounts`,
  `settings`, `events`)
- the partial unique index
  `uq_schedule_slot_at_active ON schedule(slot_at) WHERE status <> 'skipped'`
- RLS read policies for the dashboard
- private `audio` and `video` buckets
- realtime on `jobs`

### 5. Remotion

```bash
cd agent-workspace/remotion-base
npm install
npx remotion browser ensure
npm run render:example
```

`out/example.mp4` is a silent 1080×1920 render. The worker reuses this base
and injects per-job props.

For a real audio test without Supabase, Postgres, or the worker queue:

```bash
cd apps/api
.venv\Scripts\python -m scripts.smoke_local path\to\audio.wav \
  --out ..\..\agent-workspace\remotion-base\out\smoke.mp4
```

The verified local run produced 1080×1920, 30fps H.264/AAC from an 11.22s WAV.
PyAV is pinned below 16 because faster-whisper 1.2.1 still passes
`metadata_errors`, which PyAV 16+ removed.

For the first YouTube publish test, set `YOUTUBE_PRIVACY_STATUS=unlisted` (or
`private`), verify the upload, then switch back to `public`.

### 6. Run the whole stack

Install web dependencies once, then use the root scripts:

```bash
pnpm install

# Frontend + API + worker in one terminal
pnpm dev

# Or run one side at a time
pnpm dev:frontend   # Next.js only
pnpm dev:backend    # FastAPI + worker + scheduler
```

`dev:backend` starts both backend processes; `dev:frontend` starts only
Next.js. This is the recommended local flow because Ctrl+C stops the whole
process tree, including Uvicorn's reloader and Next's child server.

You can also use npm for the same scripts:

```bash
npm run dev:frontend
npm run dev:backend
```

If you prefer separate terminals, the equivalent commands are:

```bash
# API
cd apps/api
.venv\Scripts\python -m uvicorn app.main:app --reload --port 8000

# Worker + scheduler
cd apps/api
.venv\Scripts\python -m app.worker.main

# Dashboard
pnpm --filter web dev
```

Health check: `http://localhost:8000/api/health` reports the JWT mode,
Whisper model, and agent runner.

- Landing page: `http://localhost:3000/`
- Dashboard: `http://localhost:3000/dashboard`
- Set `WEB_PORT` or `API_PORT` to override the defaults (`3000` / `8000`).
- If port `3000` is already in use, stop the old Next dev server before running
  the frontend again. Next.js 16 refuses a second dev server for the same app
  directory; the runner prints this preflight error instead of killing the API
  and worker.

Dashboard auth:

- Set `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` in
  `apps/web/.env.local` to show the login gate. Create the operator user in
  Supabase → Authentication → Users; `ALLOWED_EMAIL` on the API restricts the
  token to that address.
- Leave both values empty for local mode. In that mode the dashboard calls the
  API without a token, so `AUTH_DISABLED=true` must be set on the API (default
  in development only).
- Video previews use a short-lived signed download URL from
  `GET /api/jobs/{job_id}/video-url`.
- The dashboard refreshes data by polling; Supabase Realtime is not required
  for the current flow.

## How a job moves

```
upload → queued → transcribing → generating → rendering → verifying
       → ready → awaiting_approval (default) → scheduled → publishing → published
```

Failure path: retry while `attempts < MAX_ATTEMPTS`, reusing
`jobs.transcript` when it already has words (Whisper is skipped), otherwise
`failed` + email. `needs_attention` covers manual review and skipped slots.

Publishing order:

1. YouTube upload + status receipt
2. Instagram container + processing poll + publish
3. Mark schedule published, set `delete_after = now + RETENTION_DAYS`
4. Cleanup deletes both storage objects only after both platforms publish

## Security boundaries

- Whisper runs in a dedicated `ProcessPoolExecutor`, never on the API/worker
  event loop.
- Agent and Remotion run as subprocesses with a strict env whitelist. They never
  receive Supabase, database, OAuth, or token-encryption secrets.
- `AGENT_RUN_AS_USER=agent` demotes the agent on Linux. The worker must be able
  to `runuser`/`chown` (Docker worker runs as root). Production recommendation:
  a separate Railway service for agent + render.
- Per-job workspace `agent-workspace/jobs/<job_id>/` is deleted after every
  attempt, including failures. Set `KEEP_AGENT_WORKSPACE=true` only for local
  debugging.
- Production startup guard:
  `ENVIRONMENT=production` + `AUTH_DISABLED=true` → crash; missing JWT mode,
  token-encryption key, or Supabase storage config also refuses to boot.
- `AUTH_DISABLED=false` requires a valid Supabase JWT; `ALLOWED_EMAIL` restricts
  access to one user.

## Docker

Build from the repo root; Chromium is downloaded at build time, never at
runtime:

```bash
docker build -f apps/api/Dockerfile -t motion-reels .
```

API:

```bash
docker run --env-file .env -p 8000:8000 motion-reels \
  uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Worker (default command):

```bash
docker run --env-file .env motion-reels
```

## Verification

```bash
# Python
cd apps/api
.venv\Scripts\python -m compileall app scripts
.venv\Scripts\python -m pytest

# Web
pnpm --filter web check-types
pnpm --filter web build
pnpm --filter web lint

# Silent Remotion example
cd agent-workspace/remotion-base
npm run render:example

# Real audio -> transcript -> render -> ffprobe
cd apps/api
.venv\Scripts\python -m scripts.smoke_local path\to\audio.wav \
  --out ..\..\agent-workspace\remotion-base\out\smoke.mp4 --concurrency 2
```

## Current limitations

- Dashboard UI is wired end-to-end but requires a real Supabase project for
  login, signed uploads, database reads, and signed video previews. Without
  Supabase browser env, the dashboard runs in local mode and the API must have
  `AUTH_DISABLED=true`.
- The dashboard polls the API every 10-20 seconds; Supabase Realtime is not
  wired yet.
- Real publishing requires Google/Meta credentials and a connected account.
- The full queued 6-file batch, Docker image, and Railway/Vercel deployment are
  still pending.
- `fallback` renderer produces a deterministic kinetic-caption template; switch
  `AGENT_RUNNER` to `opencode` and configure `AGENT_COMMAND` for creative
  layouts.
- Brand fonts/colors are placeholders until the operator provides final assets.

See `plan.md` for the full phase plan and locked implementation corrections.
