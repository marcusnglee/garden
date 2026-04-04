a design document for the bird alarm

> **For Claude Code:** This doc is meant to be consumed in full before writing any code. Follow the tech choices, file structure, and implementation notes exactly. Where behavior is unspecified, prefer simplicity over completeness.

---

## 1. What We're Building

A two-part system:

1. **A Python backend** that polls an Are.na channel for uploaded bird audio files, verifies and normalizes them, and serves a single daily bird call via a REST API.
2. **An iOS Shortcuts prototype** (documented here, configured manually by the user) that hits the API at alarm time and plays the day's bird call.

**Out of scope for v1:** BirdNET classification, S3/cloud storage, authentication, strict mode UI, user accounts.

---

## 2. Tech Stack

|Layer|Choice|Reason|
|---|---|---|
|Language|Python 3.11+|FFmpeg bindings, fast iteration|
|Web framework|FastAPI|Async, auto-docs, minimal boilerplate|
|Scheduler|APScheduler (AsyncIOScheduler)|In-process, no Redis needed for v1|
|Database|SQLite via `aiosqlite`|Zero-config, good enough for v1|
|Audio processing|FFmpeg via `subprocess`|Industry standard, handles any input format|
|Are.na client|`httpx` (async)|Direct HTTP calls to Are.na REST API|
|Storage|Local filesystem (`./data/audio/`)|Simple for v1; swap for S3 later|

**Install requirements:**

```
fastapi
uvicorn[standard]
apscheduler
aiosqlite
httpx
python-dotenv
```

FFmpeg must be installed on the host machine (`brew install ffmpeg` on Mac, `apt install ffmpeg` on Linux).

---

## 3. Directory Structure

```
birds-alarm/
├── main.py                  # FastAPI app + APScheduler setup
├── config.py                # Settings from env vars
├── database.py              # SQLite setup + queries
├── arena.py                 # Are.na API client
├── processor.py             # Audio verification + normalization (FFmpeg)
├── scheduler.py             # Polling job logic
├── data/
│   ├── birds.db             # SQLite database (gitignored)
│   └── audio/               # Processed .m4a files (gitignored)
│       └── processed/
├── .env                     # Local config (gitignored)
├── .env.example
└── requirements.txt
```

---

## 4. Environment Variables (`.env.example`)

```bash
# Are.na channel slug (the part after are.na/channel/)
ARENA_CHANNEL_SLUG=bird-calls-alarm

# Are.na personal access token (optional — needed only for private channels)
# Get from: https://www.are.na/developers (create an OAuth application, copy the Personal Access Token)
# WARNING: never include this in client-side code
ARENA_ACCESS_TOKEN=

# How often to poll Are.na for new uploads (in minutes)
POLL_INTERVAL_MINUTES=60

# Audio normalization settings
TARGET_LUFS=-16           # EBU R128 loudness target. -16 is comfortable alarm volume.
FADE_IN_DURATION=60       # Seconds to fade in from silence (0 = instant/strict mode)
MAX_DURATION_SECONDS=180  # Trim files longer than this

# Server
PORT=8000
```

---

## 5. Database Schema

Single table. Create on startup.

```sql
CREATE TABLE IF NOT EXISTS bird_blocks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    arena_block_id TEXT UNIQUE NOT NULL,    -- Are.na block ID (string)
    arena_title TEXT,                        -- Block title if set
    uploaded_by TEXT,                        -- Are.na username
    source_url TEXT NOT NULL,                -- Original audio URL from Are.na
    local_path TEXT,                         -- Absolute path to processed .m4a, NULL if not yet processed
    status TEXT NOT NULL DEFAULT 'pending',  -- pending | processing | ready | failed
    failure_reason TEXT,                     -- Human-readable error if status=failed
    created_at TEXT NOT NULL,                -- ISO8601 timestamp
    processed_at TEXT                        -- ISO8601 timestamp when processing completed
);

CREATE TABLE IF NOT EXISTS daily_selection (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    date TEXT UNIQUE NOT NULL,               -- YYYY-MM-DD
    arena_block_id TEXT NOT NULL,            -- FK to bird_blocks
    FOREIGN KEY (arena_block_id) REFERENCES bird_blocks(arena_block_id)
);
```

---

## 6. Are.na API Integration (`arena.py`)

**API version note:** Are.na's V2 API (`api.are.na/v2`) is officially deprecated. The current version is V3 (`api.are.na/v3`). Both appear to be live as of early 2026, but use V3 for all new code. The endpoint paths and response shapes below reflect what's known about V3; if a call fails unexpectedly, try the equivalent V2 path as a fallback.

**Base URL:** `https://api.are.na/v3`

**OAuth endpoints (for reference — not needed if using a Personal Access Token):**

- Authorization: `https://www.are.na/oauth/authorize?client_id=...&redirect_uri=...&response_type=code&scope=read`
- Token exchange: `POST https://api.are.na/v3/oauth/token`

**Relevant endpoint:**

```
GET /v3/channels/{slug}/contents?per=100&page=1
```

**Response shape (relevant fields):**

```json
{
  "contents": [
    {
      "id": 12345678,
      "class": "Attachment",
      "title": "chickadee-vermont.mp3",
      "attachment": {
        "url": "https://d2w9rnfcy7mm78.cloudfront.net/...",
        "content_type": "audio/mpeg",
        "file_size": 2048000
      },
      "user": { "username": "marcusnglee" },
      "created_at": "2026-03-30T12:00:00.000Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 100,
    "total_pages": 3,
    "total_count": 247,
    "next_page": 2,
    "prev_page": null,
    "has_more_pages": true
  }
}
```

**Implementation notes:**

- Filter for `block["class"] == "Attachment"` AND `block["attachment"]["content_type"].startswith("audio/")`
- Use `str(block["id"])` as the `arena_block_id` — Are.na returns integers
- **Pagination:** use `meta["has_more_pages"]` to decide whether to fetch the next page (cleaner than comparing `len(contents) == per`)
- Add a **200–500ms delay between paginated requests** to be a polite API citizen
- If `ARENA_ACCESS_TOKEN` is set, send as `Authorization: Bearer {token}` header; otherwise make unauthenticated requests (works for public channels)
- **Rate limiting:** monitor `X-RateLimit-Limit` and `X-RateLimit-Reset` response headers. On a `429` response, wait until the reset timestamp and retry with exponential backoff — do not retry in a tight loop
- Return a list of raw block dicts; let `scheduler.py` handle deduplication against the DB

---

## 7. Audio Processing (`processor.py`)

All processing is done with FFmpeg via `subprocess.run`. Never use `ffmpeg-python` wrapper — call FFmpeg directly for clarity.

### Step 1: Verify it's valid audio

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 {input_path}
```

If this errors or returns no duration, mark as `failed` with `failure_reason="not_valid_audio"`.

If duration is 0 or < 1 second, mark as `failed` with `failure_reason="too_short"`.

### Step 2: Normalize and process

Run a single FFmpeg command that does everything:

```bash
ffmpeg -y \
  -i {input_path} \
  -af "afade=t=in:st=0:d={FADE_IN_DURATION},loudnorm=I={TARGET_LUFS}:TP=-1.5:LRA=11" \
  -t {MAX_DURATION_SECONDS} \
  -c:a aac \
  -b:a 128k \
  -ar 44100 \
  {output_path}
```

**Filter chain breakdown:**

- `afade=t=in:st=0:d=60` — fade in from silence over 60 seconds
- `loudnorm=I=-16:TP=-1.5:LRA=11` — EBU R128 loudness normalization (two-pass is better but single-pass is fine for v1)
- `-t 180` — hard trim at 3 minutes
- `-c:a aac -b:a 128k` — AAC output, iOS-native, small file size

**Output path:** `./data/audio/processed/{arena_block_id}.m4a`

**If FADE_IN_DURATION is 0** (strict mode), drop the `afade` filter:

```bash
-af "loudnorm=I={TARGET_LUFS}:TP=-1.5:LRA=11"
```

### Error handling

- Wrap the subprocess call in try/except
- On non-zero returncode, capture stderr and store first 500 chars as `failure_reason`
- Delete any partial output file if processing failed

---

## 8. Scheduler (`scheduler.py`)

The poll job runs every `POLL_INTERVAL_MINUTES`. Logic:

```
1. Fetch all audio blocks from Are.na (paginated)
2. For each block:
   a. If arena_block_id already in DB → skip
   b. Insert with status='pending'
3. Query DB for all status='pending' records
4. For each pending record:
   a. Download source_url to a temp file (./data/audio/tmp/{id}.tmp)
   b. Set status='processing'
   c. Run processor.process(tmp_path, output_path)
   d. On success: set status='ready', local_path=output_path, processed_at=now()
   e. On failure: set status='failed', failure_reason=...
   f. Delete temp file
5. Log summary: N new blocks found, N processed, N failed
```

Also run a `select_daily` job at **3:00 AM** local time (configurable) that:

1. Checks if today's date already has a selection in `daily_selection` → if yes, done
2. Queries all `status='ready'` blocks not already used in `daily_selection`
3. Picks one at random
4. Inserts into `daily_selection`
5. If no ready blocks exist, log a warning (API will return 404 until one is available)

---

## 9. API Endpoints (`main.py`)

### `GET /today`

Returns today's bird call.

**Response (200):**

```json
{
  "date": "2026-03-30",
  "title": "chickadee-vermont",
  "uploaded_by": "marcusnglee",
  "audio_url": "http://localhost:8000/audio/12345678.m4a"
}
```

**Response (404):** `{"detail": "No bird call available for today"}` — this happens if no files are ready yet.

`audio_url` should use the request's base URL so it works both locally and when deployed. Use FastAPI's `Request` object: `str(request.base_url)`.

---

### `GET /audio/{filename}`

Serves the processed `.m4a` file.

Use FastAPI's `FileResponse` with `media_type="audio/mp4"`. Validate that `filename` matches `{arena_block_id}.m4a` pattern and exists in `./data/audio/processed/` — do not allow path traversal.

---

### `GET /library`

Returns all ready blocks (for future UI / browsing).

```json
{
  "total": 12,
  "birds": [
    { "arena_block_id": "12345678", "title": "...", "uploaded_by": "...", "processed_at": "..." }
  ]
}
```

---

### `GET /health`

```json
{ "status": "ok", "ready_count": 12, "pending_count": 2 }
```

---

## 10. Startup Sequence (`main.py`)

On app startup (`@asynccontextmanager` lifespan):

1. Create `./data/audio/processed/` and `./data/audio/tmp/` directories if they don't exist
2. Run `database.init()` to create tables
3. Start APScheduler
4. Add poll job: `IntervalTrigger(minutes=POLL_INTERVAL_MINUTES)`
5. Add daily selection job: `CronTrigger(hour=3, minute=0)`
6. **Run the poll job once immediately** at startup (so fresh installs don't wait an hour)

---

## 11. iOS Shortcuts Prototype

This is configured manually by the user in the iOS Shortcuts app. Document these steps in a `SHORTCUTS_SETUP.md` in the repo.

### Prerequisites

- The backend must be running and accessible from the phone (either locally via same Wi-Fi, or deployed to a public URL like Fly.io / Railway)
- At least one bird call must be processed and ready (`GET /health` should show `ready_count > 0`)

### Creating the Shortcut

In the **Shortcuts** app → **+** (New Shortcut):

1. **Add action: "Get Contents of URL"**
    
    - URL: `http://YOUR_SERVER_IP:8000/today`
    - Method: GET
2. **Add action: "Get Dictionary Value"**
    
    - Key: `audio_url`
    - Dictionary: (output of previous step)
3. **Add action: "Get Contents of URL"**
    
    - URL: (output of previous step — the `audio_url` value)
    - Method: GET
4. **Add action: "Play Sound"**
    
    - Sound: (output of previous step — the downloaded audio file)
5. Name the shortcut **"Birds Alarm"**
    

### Setting the Automation

In the **Automation** tab → **+** → **Personal Automation** → **Time of Day**:

- Time: your alarm time (e.g. 7:00 AM)
- Repeat: Daily
- Run Shortcut: "Birds Alarm"
- **Disable "Ask Before Running"** — this is required for it to fire without tapping a notification

### Known Limitations

- iOS may not play audio if the device is on silent/DND. User should set their alarm time shortcut to also run "Set Volume" before "Play Sound".
- If the server is unreachable, the shortcut will silently fail. For v1 this is acceptable.
- The Shortcuts audio player shows a system now-playing widget but doesn't prevent the screen from locking.

---

## 12. Running Locally

```bash
# Install dependencies
pip install -r requirements.txt

# Copy and fill in env
cp .env.example .env
# Edit .env: set ARENA_CHANNEL_SLUG at minimum

# Run
uvicorn main:app --reload --port 8000

# Verify
curl http://localhost:8000/health
curl http://localhost:8000/today
```

---

## 13. What's Explicitly Deferred

These are real requirements from the original spec but out of scope for this build:

- **BirdNET classification** — verify it's a bird sound, not just audio. Requires torch + model (~500MB). Add as an optional flag later.
- **S3/R2 storage** — swap `local_path` for a CDN URL when deploying at scale.
- **Strict mode** — a separate `/today?mode=strict` endpoint that serves a file processed with `FADE_IN_DURATION=0`. Schema and processor already support this.
- **Are.na upload naming convention** — enforce `CommonName_Location_Date.ext` and parse for richer metadata.
- **Native iOS app** — the Shortcuts prototype is sufficient for personal use. Build the app if you want App Store distribution or a more reliable background trigger.
- **Authentication** — the API is currently open. Add an API key header if exposing publicly.