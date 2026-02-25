# InfiniteDJ

An infinite, automatic DJ live stream for YouTube. Audio clips are connected as a graph — each clip ends on a song that the next clip begins with, allowing seamless concatenation. The server selects the next clip dynamically based on genre schedule, BPM compatibility, and play history.

**Graph example:**
- Clip 1: A→B, Clip 2: B→C, Clip 3: C→A
- Clip 1 ends at the end of song B, Clip 2 starts at the beginning of song B → perfect cut by pure concatenation.

---

## Tech stack

| Component | Technology |
|---|---|
| Main server | **Ktor** (Kotlin, Netty engine) |
| Dependency injection | **Koin** |
| ORM / DB | **Exposed** + **SQLite** |
| Serialization | kotlinx.serialization |
| Time-stretching | **Rubber Band** (via Python worker) |
| Streaming | **FFmpeg** → RTMP → YouTube Live |
| Admin frontend | **React** (Vite) + **React Flow** |
| Dev infra | **Docker Compose** (Windows) |
| Prod infra | **Raspberry Pi** (ARM64, SSD 128 GB) |

---

## Project structure

```
infinitedj/
├── docker-compose.yml
├── server/                 # Ktor (this module)
│   ├── Dockerfile
│   └── src/
├── worker/                 # Python — time-stretching via Rubber Band
│   ├── Dockerfile
│   └── stretch.py
├── admin/                  # React (Vite) — built to static, served by Ktor
│   ├── src/
│   └── dist/               # served via Ktor Static Content plugin
├── clips/                  # audio files shared between containers
└── data/                   # SQLite DB volume
```

---

## Database schema

```sql
CREATE TABLE clips (
  id          INTEGER PRIMARY KEY,
  file        TEXT NOT NULL,
  starts_with TEXT NOT NULL,   -- name of the opening song
  ends_with   TEXT NOT NULL,   -- name of the closing song
  genre       TEXT NOT NULL,
  bpm_start   REAL,            -- BPM at the start of the clip (after stretch)
  bpm_end     REAL,            -- BPM at the end of the clip (after stretch)
  added_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE schedule (
  id        INTEGER PRIMARY KEY,
  from_time TIME NOT NULL,
  to_time   TIME NOT NULL,
  genre     TEXT NOT NULL      -- e.g. "house", "techno", "ambient"
);
```

---

## BPM & time-stretching

Clips are pre-stretched at upload time (not at runtime) using Rubber Band via the Python worker. The BPM glides progressively across the clip (e.g. 150→160 BPM) so the transition into the next clip is smooth. The Raspberry Pi only streams pre-rendered files via FFmpeg — no processing load during the live.

Upload workflow:
```
Upload clip → Python worker (Rubber Band) → stretched file saved to /clips → metadata written to DB → Ktor streams
```

> `bpm_end` of a clip must match `bpm_start` of the next clip. The worker can re-stretch a clip with a few clips of advance once the next one is known.

---

## Development plan

### Phase 1 — Database & core models
- [ ] Remove Ktor generator placeholder files (`HelloService.kt`, `UsersSchema.kt`, `Frameworks.kt`)
- [ ] Create `Clip` Exposed table
- [ ] Create `Schedule` Exposed table
- [ ] Wire Exposed + SQLite in `Application.kt` (file path: `data/infinitedj.db`)
- [ ] Basic health-check route `GET /health`

### Phase 2 — Clip management REST API
- [ ] `GET /api/clips` — list all clips
- [ ] `POST /api/clips` — upload audio file + metadata (multipart form)
- [ ] `DELETE /api/clips/{id}` — remove a clip
- [ ] `GET /api/clips/{id}/next` — return eligible next clips (debug)
- [ ] `GET /api/schedule` — list schedule entries
- [ ] `POST /api/schedule` — create a schedule entry
- [ ] `DELETE /api/schedule/{id}` — remove a schedule entry

### Phase 3 — Python worker (time-stretching)
- [ ] Create `worker/` with `Dockerfile` (Python + Rubber Band + soundfile)
- [ ] Implement `stretch.py`: progressive BPM stretch, writes `*_stretched.wav`
- [ ] Add `docker-compose.yml` with `ktor-app` and `python-worker` sharing `./clips`
- [ ] Ktor triggers the worker via HTTP after clip upload (worker exposes `/stretch`)

### Phase 4 — Stream Engine
- [ ] `ClipSelector`: query DB for clips where `starts_with = previous.ends_with`, filtered by schedule genre, excluding recently played
- [ ] `StreamEngine`: coroutine loop — selects next clip, hands off to FFmpeg
- [ ] `FfmpegRunner`: launches FFmpeg process to stream WAV via RTMP to YouTube
- [ ] Queue next clip a few seconds before current one ends
- [ ] Inject `StreamEngine` via Koin, start on application startup

### Phase 5 — WebSocket real-time state
- [ ] `GET /ws/state` WebSocket endpoint
- [ ] Broadcast on clip change: `{ currentClip, nextClip, startedAt, genre }`
- [ ] Broadcast on stream status: `{ streaming: true/false }`

### Phase 6 — React admin frontend
- [ ] Init `admin/` with Vite + React + TypeScript
- [ ] Install React Flow
- [ ] Graph view: nodes = songs, edges = clips; click edge → clip details
- [ ] Upload form: audio file + `starts_with`, `ends_with`, `genre`, `bpm_start`, `bpm_end`
- [ ] Schedule manager: time range + genre (add / delete)
- [ ] Live status bar: current clip, genre, WebSocket indicator
- [ ] Build output → `server/src/main/resources/static/`

### Phase 7 — Docker & deployment
- [ ] Finalize `docker-compose.yml` for local dev
- [ ] `server/Dockerfile`: multi-stage (Gradle → fat JAR → JRE runtime)
- [ ] `worker/Dockerfile`: Python 3.12 slim + Rubber Band + soundfile
- [ ] Test full stack with `docker compose up`
- [ ] Cross-compile for ARM64 with `docker buildx` for Raspberry Pi
- [ ] Document Pi deploy (SSD mount, YouTube RTMP key env var)

---

## Known open points

- Clips that change BPM internally (not just at junctions) — not handled yet
- YouTube Content ID on live streams — copyright issue unresolved
- Stream monitoring: crash detection + automatic restart
- Play history persistence for anti-repetition logic

---

## Running locally (no Docker)

```bash
./gradlew run   # Ktor on :8080
```

## Running with Docker Compose

```bash
docker compose up --build
```

Admin UI at `http://localhost:8080/admin`.
