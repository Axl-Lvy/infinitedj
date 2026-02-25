# InfiniteDJ — Claude context

## What this project is

An infinite YouTube DJ live stream. Pre-recorded audio clips are nodes in a graph. Each clip starts at the beginning of a song and ends at the end of another. The `ends_with` song of clip N is always the `starts_with` song of clip N+1 — transitions are seamless because clips are simply concatenated (no crossfade needed).

## Ktor plugins in use (Gradle Kotlin DSL, Netty engine)

- `Routing`
- `ContentNegotiation` + `kotlinx.serialization` (JSON)
- `StaticContent` — serves the React build from `admin/dist/` (mounted at `/admin`)
- `WebSockets` — streams live state to the admin UI
- `Exposed` — ORM for SQLite
- `StatusPages` — global error handling
- `Koin` — dependency injection

## Docker Compose services

```yaml
services:
  ktor-app:
    build: ./server
    ports: ["8080:8080"]
    volumes:
      - ./clips:/clips
      - ./data:/data        # SQLite DB lives here

  python-worker:
    build: ./worker
    volumes:
      - ./clips:/clips      # reads source clips, writes stretched output
```

The worker exposes a simple HTTP endpoint `/stretch` that Ktor calls after a clip upload.

## Stream Engine — detailed flow

Runs as a Kotlin coroutine injected via Koin, started on application startup:

1. Current clip finishes (FFmpeg process exits)
2. Query DB: `SELECT * FROM clips WHERE starts_with = :prevEndsWidth AND genre = :activeGenre AND id NOT IN (:recentlyPlayed)`
3. Pick best candidate (random among eligible, or by some scoring)
4. Launch new FFmpeg process: `ffmpeg -re -i /clips/<file> -f flv rtmp://...`
5. Broadcast new state via WebSocket to admin UI
6. Repeat

Start queuing the next clip selection ~10 seconds before the current one ends to avoid gaps.

## Python worker — Rubber Band stretch implementation

```python
import rubberband
import numpy as np
import soundfile as sf

audio, sr = sf.read("clip.wav")
n_frames = len(audio)

stretcher = rubberband.RubberBandStretcher(sr, channels=2)
stretcher.setMaxProcessSize(1024)

output = []
block_size = 1024

for i in range(0, n_frames, block_size):
    progress = i / n_frames
    ratio = np.interp(progress, [0, 1], [bpm_start / bpm_end, 1.0])
    stretcher.setTimeRatio(ratio)
    block = audio[i:i + block_size]
    stretcher.process(block, final=(i + block_size >= n_frames))
    available = stretcher.available()
    if available > 0:
        output.append(stretcher.retrieve(available))

result = np.concatenate(output)
sf.write("clip_stretched.wav", result, sr)
```

`bpm_end` target is only known once the next clip is chosen. The worker re-stretches the current clip with a few clips of advance. Stretched files are stored as `<id>_stretched.wav` alongside the originals in `/clips`.

## File naming conventions

- Raw uploaded clip: `/clips/<id>.wav`
- Stretched clip: `/clips/<id>_stretched.wav`
- SQLite DB: `/data/infinitedj.db`
- React static build: `server/src/main/resources/static/` (served at `/admin`)

## WebSocket message shapes

```json
// clip change
{ "type": "clip_change", "currentClip": { "id": 1, "file": "...", "genre": "house" }, "nextClip": { ... }, "startedAt": "2025-01-01T20:00:00Z" }

// stream status
{ "type": "stream_status", "streaming": true }
```

## Key architectural constraints

- **No runtime audio processing on the Pi** — all stretching is done at upload time by the Python worker
- **Transitions are graph-based, not time-based** — `ends_with` must exactly equal `starts_with` of the next clip; there is no crossfade
- **BPM fields in DB reflect the stretched file**, not the original
- **The admin frontend is a static SPA** — built separately with Vite, copied into Ktor's resources, no SSR
- **SQLite only** — no Postgres, keep it simple for the Pi

## Deployment target

Raspberry Pi 4 (ARM64), SSD 128 GB external. Docker images cross-compiled from Windows with `docker buildx`. YouTube RTMP key passed as environment variable. Expected clip library: ~500 clips, 15–25 GB total.
