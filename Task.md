# task.md — Motif
**Project:** Motif — AI Music Video Generator  
**Version:** 1.0  
**Sprint Model:** 3-day rapid prototype → iterative V1 build  
**Last Updated:** April 2026  

---

## How to Use This File

- Tasks are organized by **Phase** (1 = prototype, 2 = V1 polish, 3 = V2 features)
- Each task has a unique **Task ID**, **estimate**, **priority**, and **dependencies**
- Status column: `[ ]` Not started · `[~]` In progress · `[x]` Done · `[!]` Blocked
- Update this file as you work — it is the single source of truth for project progress

---

## Quick Reference — Task Summary

| Phase | Tasks | Est. Total Time |
|-------|-------|----------------|
| Phase 1 — Core prototype | 24 tasks | ~3 days |
| Phase 2 — V1 polish | 18 tasks | ~5 days |
| Phase 3 — V2 features | 22 tasks | ~3 weeks |

---

## Phase 1 — Core Prototype
> **Goal:** A working end-to-end pipeline. Song in → MP4 reel out. No UI, no production polish. Verifiable via `curl` and a terminal.

### EPIC-01 · Project Setup

---

#### TASK-001 · Initialize repository structure
- **Status:** `[ ]`
- **Priority:** P0 — blocker for everything
- **Estimate:** 30 min
- **Assigned:** -
- **Dependencies:** None

**Subtasks:**
- [ ] Create GitHub repository named `motif`
- [ ] Initialize with MIT `LICENSE` file
- [ ] Create root `.gitignore` (Python, Node.js, `.env`, `__pycache__`, `.next`, `/tmp/motif`)
- [ ] Create directory structure:
  ```
  motif/
  ├── backend/
  │   └── pipeline/
  ├── frontend/
  │   ├── app/
  │   └── components/
  └── docs/
  ```
- [ ] Create `.env.example` with all required variables (no values)
- [ ] Create empty `README.md` with project name and placeholder sections
- [ ] Make initial commit: `chore: initialize repository`

**Definition of Done:** `git clone` of the repo produces the expected directory tree with no errors.

---

#### TASK-002 · Configure Python backend environment
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 45 min
- **Dependencies:** TASK-001

**Subtasks:**
- [ ] Create `backend/requirements.txt` with all dependencies (see requirements.md §7)
- [ ] Create `backend/Dockerfile`:
  - Base image: `python:3.11-slim`
  - Install FFmpeg via `apt-get`
  - Install Python dependencies
  - Copy source
  - Expose port 8000
- [ ] Create `backend/config.py` — loads all env vars with defaults and type validation
- [ ] Create `backend/models.py` — all Pydantic schemas (Job, APMOutput, LIEOutput, Scene, etc.)
- [ ] Verify `uvicorn` starts with no import errors: `uvicorn main:app --reload`

**Definition of Done:** `docker build ./backend` succeeds. `uvicorn` starts and `GET /api/v1/health` returns 200.

---

#### TASK-003 · Configure Next.js frontend environment
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 30 min
- **Dependencies:** TASK-001

**Subtasks:**
- [ ] Scaffold frontend: `npx create-next-app@latest frontend --typescript --tailwind --app`
- [ ] Install additional dependencies: `npm install socket.io-client`
- [ ] Create `frontend/Dockerfile`:
  - Base image: `node:20-alpine`
  - Copy and install deps
  - Expose port 3000
- [ ] Create `frontend/lib/api.ts` — typed API client with base URL from env
- [ ] Set `NEXT_PUBLIC_API_URL` in `.env.example`

**Definition of Done:** `npm run dev` starts frontend on port 3000. Browser shows Next.js default page.

---

#### TASK-004 · Create docker-compose.yml
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 20 min
- **Dependencies:** TASK-002, TASK-003

**Subtasks:**
- [ ] Define `backend` service: build `./backend`, port 8000, env_file `.env`, volume for `/tmp/motif`
- [ ] Define `frontend` service: build `./frontend`, port 3000, depends_on backend
- [ ] Define shared volume: `motif_tmp`
- [ ] Test: `docker-compose up --build` starts both services successfully
- [ ] Test: frontend at `http://localhost:3000` and backend health at `http://localhost:8000/api/v1/health`

**Definition of Done:** `docker-compose up` starts both services. Both URLs return expected responses.

---

### EPIC-02 · FastAPI Backend Shell

---

#### TASK-005 · Create FastAPI application shell with health endpoint
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 45 min
- **Dependencies:** TASK-002

**Subtasks:**
- [ ] Create `backend/main.py` with FastAPI app instantiation
- [ ] Configure CORS middleware (allow configured frontend origin)
- [ ] Add startup event: check FFmpeg is installed, check Whisper can be imported
- [ ] Implement `GET /api/v1/health` endpoint returning:
  - `status`: ok / degraded
  - `whisper_model`: value from config
  - `llm_provider`: claude / ollama
  - `ollama_available`: bool (ping Ollama `/api/tags`)
  - `image_provider`: replicate / local
  - `gpu_available`: bool (torch.cuda.is_available())
  - `ffmpeg_version`: parsed from `ffmpeg -version`
- [ ] Add global exception handler returning structured JSON errors

**Definition of Done:** `GET /api/v1/health` returns accurate system state. FFmpeg absence returns `status: degraded` with clear message.

---

#### TASK-006 · Implement job creation endpoint
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-005

**Subtasks:**
- [ ] Implement `POST /api/v1/jobs` with `multipart/form-data` parsing
- [ ] Validate file: size ≤ 50MB, accepted MIME types only (use `python-magic`)
- [ ] Validate `duration_s` parameter: must be 30, 45, 60, or 90
- [ ] Generate UUID job ID: `j_{uuid4().hex[:8]}`
- [ ] Save uploaded file to `/tmp/motif/{job_id}/audio_original.{ext}` using UUID-based path
- [ ] Create in-memory job store (dict for prototype — Redis in V2)
- [ ] Create initial `Job` record with status `queued`
- [ ] Return 202 response with job record
- [ ] Trigger pipeline as a background task (`BackgroundTasks`)

**Definition of Done:**
```bash
curl -X POST localhost:8000/api/v1/jobs -F "file=@song.mp3" -F "duration_s=60"
# Returns {"job_id": "j_abc123", "status": "queued", ...}
```

---

#### TASK-007 · Implement job status endpoint and WebSocket
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1 hour
- **Dependencies:** TASK-006

**Subtasks:**
- [ ] Implement `GET /api/v1/jobs/:id` returning current job state
- [ ] Return 404 with `JOB_NOT_FOUND` for unknown job IDs
- [ ] Implement WebSocket endpoint: `ws://host/api/v1/jobs/:id/events`
- [ ] Create `backend/events.py` — event emitter that broadcasts to all WebSocket connections for a job
- [ ] Implement `GET /api/v1/jobs/:id/download` — stream MP4 file, return 409 if not complete
- [ ] Write utility `emit_event(job_id, event_type, payload)` called from pipeline stages

**Definition of Done:** WebSocket connection receives events as the pipeline progresses. Download endpoint streams MP4 correctly.

---

### EPIC-03 · Audio Processing Module (APM)

---

#### TASK-008 · Implement Whisper transcription
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 2 hours
- **Dependencies:** TASK-006

**Subtasks:**
- [ ] Create `backend/pipeline/apm.py`
- [ ] Implement `transcribe(audio_path: str, model_name: str) -> dict`
  - Load Whisper model (use module-level singleton to avoid reloading)
  - Run transcription with `word_timestamps=True`
  - Extract word list with `start_ms`, `end_ms`, `word`, `segment_index`
  - Extract detected language and confidence
- [ ] Implement fallback: if model `large-v3` fails to load (insufficient VRAM), automatically fall back to `base`
- [ ] Log transcription duration and word count
- [ ] Write unit test: transcribe a 10-second reference audio clip, assert word timestamps are present

**Definition of Done:** `transcribe("test_song.mp3", "base")` returns a dict with `lyrics` list where every item has `start_ms`, `end_ms`, and `word`.

---

#### TASK-009 · Implement beat detection and section analysis
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-008

**Subtasks:**
- [ ] Add `detect_beats(audio_path: str) -> dict` to `apm.py`
  - Use `librosa.beat.beat_track` for BPM and beat grid
  - Convert beat frame indices to milliseconds
  - Compute energy curve: RMS energy per 500ms window, normalized to 0.0–1.0
- [ ] Add `detect_sections(audio_path: str, beat_grid: list) -> list` 
  - Use energy curve inflection points to guess section boundaries
  - Label sections: intro / verse / chorus / bridge / outro (heuristic based on position and energy)
  - Fallback: if less than 3 sections detected, divide uniformly into 8s windows
- [ ] Add `process_audio(audio_path: str) -> APMOutput` — combines transcription + beat + sections
- [ ] Emit `stage_update` WebSocket event at start and completion

**Definition of Done:** `process_audio("song.mp3")` returns `APMOutput` with populated `bpm`, `beat_grid_ms`, `energy_curve`, `sections`, and `lyrics`.

---

### EPIC-04 · Lyrics Intelligence Engine (LIE)

---

#### TASK-010 · Implement Claude API scene planner
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 2 hours
- **Dependencies:** TASK-009

**Subtasks:**
- [ ] Create `backend/pipeline/lie.py`
- [ ] Write system prompt: narrative extraction + scene planning rules (see requirements FR-LIE-01)
- [ ] Implement `plan_scenes_claude(apm_output: APMOutput, duration_s: int) -> LIEOutput`
  - Build user prompt from lyrics, sections, beat grid, target duration
  - Call `claude-sonnet-4-6` via Anthropic SDK
  - Return `LIEOutput` with narrative, emotional_arc, scenes list
- [ ] Implement JSON parsing with 3-strategy fallback (direct → strip fences → find braces)
- [ ] Implement retry logic: if parse fails, retry with stricter prompt once
- [ ] Raise `LIEParseError` if both attempts fail
- [ ] Validate scene count (6–12) and timestamp range after parsing
- [ ] Emit WebSocket events: `stage_update` at start and `scene_plan_ready` on success

**Definition of Done:** `plan_scenes_claude(apm_output, 60)` returns `LIEOutput` with 6–12 scenes whose timestamps sum to ~60 seconds.

---

#### TASK-011 · Implement Ollama scene planner
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-010

**Subtasks:**
- [ ] Implement `plan_scenes_ollama(apm_output: APMOutput, duration_s: int, model: str) -> LIEOutput`
  - Same interface as Claude version
  - POST to `http://localhost:11434/api/generate` with `stream: false`
  - Set `temperature: 0.3` for JSON reliability
  - Apply same 3-strategy JSON fallback + retry as Claude version
- [ ] Add `OLLAMA_UNAVAILABLE` error with clear message if connection refused
- [ ] Add availability pre-check: `GET /api/tags` before sending generation request
- [ ] Implement `plan_scenes(apm_output, duration_s) -> LIEOutput` — router function that delegates to Claude or Ollama based on `LLM_PROVIDER` env var

**Definition of Done:** Setting `LLM_PROVIDER=ollama` with Ollama running produces equivalent output to the Claude version (structure identical, content may vary in quality).

---

#### TASK-012 · Validate and sanitize LIE output
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1 hour
- **Dependencies:** TASK-011

**Subtasks:**
- [ ] Implement `validate_scene_plan(plan: dict, target_ms: int) -> LIEOutput`
  - Ensure `scenes` list is non-empty
  - Ensure all scene IDs are unique — auto-assign if duplicates found
  - Ensure all timestamps are integers (coerce from float or string if needed)
  - Ensure all scenes have required fields — fill defaults if missing
  - Adjust last scene's `end_ms` to match target duration if off by > 3 seconds
  - Log a warning if total duration is off by > 5 seconds
- [ ] Write unit tests for each validation case

**Definition of Done:** All validation edge cases (float timestamps, missing fields, duplicate IDs, wrong total duration) are handled without crashing.

---

### EPIC-05 · Scene Generation Engine (SGE)

---

#### TASK-013 · Implement Replicate SDXL image generation
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-012

**Subtasks:**
- [ ] Create `backend/pipeline/sge.py`
- [ ] Implement `generate_image_replicate(prompt: str, negative_prompt: str, scene_id: str, output_dir: str, seed: int) -> str`
  - Call Replicate SDXL model with 1024×1792 resolution
  - Download returned image URL to local path `{output_dir}/scene_{scene_id}.png`
  - Return local file path
- [ ] Implement skip logic: return existing path if file already exists
- [ ] Handle Replicate API errors: timeout → retry once → raise `SGEGenerationError`
- [ ] Emit `scene_ready` WebSocket event after each scene image is saved

**Definition of Done:** `generate_image_replicate("cinematic shot...", ..., "s001", "/tmp/motif/j_abc", 42)` produces a 1024×1792 PNG at the expected path.

---

#### TASK-014 · Implement local SDXL image generation
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 2 hours
- **Dependencies:** TASK-013

**Subtasks:**
- [ ] Implement `_load_pipeline() -> pipe` — singleton loader for diffusers pipeline
  - GPU path: `StableDiffusionXLPipeline` with `torch.float16` and xformers
  - CPU path: `AutoPipelineForText2Image` with `sd-turbo`
  - Store in module-level `_pipe` variable — load once, reuse across all scenes
- [ ] Implement `generate_image_local(prompt, negative_prompt, scene_id, output_dir, seed, cpu_mode) -> str`
  - GPU: 1024×1792, 25 inference steps, CFG 7.5
  - CPU: 512×912, 4 inference steps, CFG 0.0 (sd-turbo)
- [ ] Implement `generate_image(...)` router — delegates to Replicate or local based on `IMAGE_PROVIDER`
- [ ] Implement `generate_all_scenes(scenes, output_dir, style_prefix) -> list[Scene]`
  - Iterate scenes sequentially
  - Call `generate_image` for each
  - Update `scene.image_path` and `scene.generation_status`
  - Emit `scene_ready` WebSocket event per scene

**Definition of Done:** `CPU_MODE=true IMAGE_PROVIDER=local` generates all scenes using sd-turbo without CUDA errors. Images saved at correct paths.

---

### EPIC-06 · Composition & Sync Engine (CSE)

---

#### TASK-015 · Generate ASS subtitle file from word timestamps
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-014

**Subtasks:**
- [ ] Create `backend/pipeline/cse.py`
- [ ] Implement `generate_subtitle_file(lyrics: list, output_path: str) -> str`
  - Write ASS header with correct style definition (font, size, position, outline)
  - For each word in lyrics: write a `Dialogue` line with start/end time and word text
  - Use karaoke-style: each word appears individually, not accumulating
  - Handle special characters in lyrics (escape `{}`, `\N`, etc.)
- [ ] Test subtitle file with `ffplay` to verify timing visually

**Definition of Done:** Subtitle file produces word-by-word captions aligned to the song when previewed with `ffplay`.

---

#### TASK-016 · Implement Ken Burns animation and FFmpeg scene assembly
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 3 hours
- **Dependencies:** TASK-015

**Subtasks:**
- [ ] Implement `compute_beat_snapped_cuts(scenes: list, beat_grid: list) -> list`
  - For each scene, find nearest beat within ±300ms
  - Snap `scene.start_ms` to that beat if within tolerance
  - Return adjusted scenes list
- [ ] Implement `build_zoompan_filter(scene: Scene, fps: int) -> str`
  - Compute number of frames from scene duration
  - Build FFmpeg `zoompan` filter string with slow zoom (1.0 → 1.05) and random pan direction
- [ ] Implement `build_ffmpeg_filter_complex(scenes: list, fps: int) -> str`
  - Concatenate all zoompan filters with `concat` filter
  - Output: single `[outv]` video stream
- [ ] Implement `assemble_reel(scenes, audio_path, subtitle_path, output_path, fps=30)`
  - Build FFmpeg command from inputs + filter complex
  - Burn in subtitles with `subtitles` filter
  - Mix original audio at full volume
  - Trim to exact target duration with `-t`
  - Run via `subprocess.run(..., check=True)`
  - Raise `CSEError` on non-zero exit code with stderr content
- [ ] Write integration test: assemble 2 test scenes, verify output is valid MP4

**Definition of Done:** `assemble_reel(...)` produces a valid 1080×1920 MP4 with Ken Burns animation, subtitles, and original audio. Duration is within ±2s of target.

---

### EPIC-07 · Pipeline Orchestration

---

#### TASK-017 · Implement pipeline orchestrator
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-016

**Subtasks:**
- [ ] Create `backend/pipeline/orchestrator.py`
- [ ] Implement `run_pipeline(job: Job) -> None` — the main background task function
  - Stage 1: Call `apm.process_audio()` → update job with APM output, emit progress
  - Stage 2: Call `lie.plan_scenes()` → update job with LIE output, emit progress
  - Stage 3: Call `sge.generate_all_scenes()` → update job with scene image paths, emit per-scene events
  - Stage 4: Call `cse.assemble_reel()` → update job with output path, emit progress
  - Stage 5: Set job status to `complete`, emit `reel_ready` event
  - On any exception: set job status to `failed`, store error, emit `error` event
- [ ] Implement `update_job(job_id, updates: dict)` — thread-safe job store update
- [ ] Clean up temp files after pipeline completes: delete `audio_original.{ext}`, delete intermediate MP4

**Definition of Done:** End-to-end run via `curl -X POST localhost:8000/api/v1/jobs -F "file=@song.mp3"` produces a downloadable reel at `GET /api/v1/jobs/:id/download`.

---

#### TASK-018 · Implement download endpoint and file cleanup scheduler
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1 hour
- **Dependencies:** TASK-017

**Subtasks:**
- [ ] Implement `GET /api/v1/jobs/:id/download`
  - Return 404 if job ID not found
  - Return 409 if job not in `complete` status
  - Stream MP4 file using `FileResponse`
  - Set `Content-Disposition: attachment; filename="motif_{job_id}.mp4"`
- [ ] Implement `GET /api/v1/jobs/:id/thumbnail` — return thumbnail JPG if it exists
- [ ] Add startup background task: every 30 minutes, scan `/tmp/motif/` and delete job directories older than 24 hours

**Definition of Done:** Download endpoint returns playable MP4. Files are cleaned up after 24 hours.

---

### EPIC-08 · Frontend — Upload Screen

---

#### TASK-019 · Build upload screen
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 2 hours
- **Dependencies:** TASK-007

**Subtasks:**
- [ ] Create `frontend/app/page.tsx` — upload screen
- [ ] Build `<AudioUploader>` component:
  - Drag-and-drop zone with dashed border
  - Click-to-browse fallback using hidden `<input type="file">`
  - Accept: `audio/mpeg, audio/wav, audio/m4a, audio/flac, audio/ogg`
  - Show file name and size after selection
  - Show error if file > 50MB before submitting
- [ ] Build duration selector: 4 buttons (30s / 45s / 60s / 90s), 60s active by default
- [ ] Build LLM provider selector: collapsible advanced settings, dropdown (Claude / Ollama)
- [ ] Build "Generate Reel" submit button: disabled until valid file selected, shows spinner on submit
- [ ] On submit: call `POST /api/v1/jobs`, redirect to `/process/[job_id]` on success

**Definition of Done:** User can drag a file, select duration, and click Generate. Browser navigates to processing screen and job is created.

---

### EPIC-09 · Frontend — Processing Screen

---

#### TASK-020 · Build processing screen with live WebSocket updates
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 2.5 hours
- **Dependencies:** TASK-019, TASK-007

**Subtasks:**
- [ ] Create `frontend/app/process/[id]/page.tsx`
- [ ] Connect to `ws://api/api/v1/jobs/:id/events` on page mount
- [ ] Build `<PipelineProgress>` component:
  - 5 step indicators: Transcription → Scene Planning → Generating Images → Composing → Ready
  - Active step shows spinner; completed steps show checkmark
  - Current step label shown below indicator
- [ ] Build `<SceneStrip>` component:
  - Horizontal scrollable row of `<SceneCard>` components
  - Each card shows: scene image thumbnail, lyric line, mood tag, duration badge
  - New cards appear via animation as `scene_ready` WebSocket events arrive
  - Show placeholder skeleton cards for scenes not yet generated
- [ ] Handle `reel_ready` event: navigate to `/result/[job_id]`
- [ ] Handle `error` event: show error banner with error message and recovery suggestion

**Definition of Done:** Processing screen shows real-time progress. Scene cards appear as images are generated. Navigation to result screen is automatic.

---

### EPIC-10 · Frontend — Result Screen

---

#### TASK-021 · Build result and download screen
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-020

**Subtasks:**
- [ ] Create `frontend/app/result/[id]/page.tsx`
- [ ] Fetch job data from `GET /api/v1/jobs/:id` on mount
- [ ] Build `<VideoPreview>` component:
  - HTML5 `<video>` element with `src` pointing to preview URL
  - `autoPlay`, `muted`, `loop`, `playsInline` attributes set
  - Display controls bar below video (play/pause, seek, unmute)
  - Constrain to 9:16 aspect ratio in a centered container
- [ ] Add "Download Reel" button: `<a href="/api/v1/jobs/:id/download" download>`
- [ ] Add "Generate Another" button: clears state and navigates to upload screen
- [ ] Display generation stats: total time, number of scenes, duration
- [ ] Show all scene cards in a row below the video player

**Definition of Done:** Result screen plays the reel in a 9:16 player. Download button triggers file download. Stats displayed correctly.

---

### EPIC-11 · End-to-End Testing

---

#### TASK-022 · End-to-end pipeline test with real song
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 2 hours
- **Dependencies:** TASK-021

**Subtasks:**
- [ ] Identify 3 test songs: English pop (clear narrative), instrumental, non-English
- [ ] Run full pipeline for each via `curl` and verify output MP4 with `ffprobe`
- [ ] Verify against acceptance criteria AT-01 through AT-10 (see requirements.md §9.1)
- [ ] Document any failures with repro steps in `docs/test-results.md`
- [ ] Fix any critical failures found before moving to Phase 2

**Definition of Done:** All 10 functional acceptance tests pass for English pop test song.

---

#### TASK-023 · Error handling tests
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 1 hour
- **Dependencies:** TASK-022

**Subtasks:**
- [ ] Test AT-11: Upload 60MB file → expect `FILE_TOO_LARGE`
- [ ] Test AT-12: Upload `.txt` renamed to `.mp3` → expect `UNSUPPORTED_FORMAT`
- [ ] Test AT-13: Stop Ollama, run with `LLM_PROVIDER=ollama` → expect `OLLAMA_UNAVAILABLE`
- [ ] Test AT-14: Manually break JSON response → expect `LIE_PARSE_FAILED`
- [ ] Test AT-15: Poll unknown job ID → expect 404

**Definition of Done:** All 5 error test cases produce the expected error codes without unhandled exceptions or 500 responses.

---

#### TASK-024 · Write README and record demo GIF
- **Status:** `[ ]`
- **Priority:** P0
- **Estimate:** 2 hours
- **Dependencies:** TASK-022

**Subtasks:**
- [ ] Complete `README.md`:
  - Add demo GIF at top (record with Loom or ScreenToGif)
  - Complete Quick Start section (5 lines max to get running)
  - Add architecture diagram (export pipeline SVG as PNG)
  - Add tech stack table
  - Add cost per run table
  - Add FAQ for common issues (Ollama not running, CUDA errors, etc.)
- [ ] Add GitHub repository topics: `ai-video`, `music-video`, `generative-ai`, `whisper`, `stable-diffusion`, `fastapi`, `nextjs`, `ffmpeg`, `anthropic`, `ollama`
- [ ] Add repository description: "Turn any song into a short-form video reel using AI"
- [ ] Make initial public release tag: `git tag v0.1.0-prototype`

**Definition of Done:** Repository is presentable on GitHub. A developer can follow README and get a reel in < 30 minutes.

---

## Phase 2 — V1 Polish
> **Goal:** Production-quality code, proper error handling, a polished UI, and documentation suitable for a portfolio or technical interview.

---

#### TASK-025 · Add proper logging throughout the pipeline
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 1 hour
- **Dependencies:** TASK-022

**Subtasks:**
- [ ] Create `backend/logger.py` using Python `logging` with structured JSON format
- [ ] Add log entries at start/end of every pipeline stage with duration
- [ ] Log all external API calls with URL, model, and response time
- [ ] Add request ID to all logs (derived from `job_id`)
- [ ] Configure log level via `LOG_LEVEL` env var (default: `INFO`)

---

#### TASK-026 · Add input file MIME type validation
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 30 min
- **Dependencies:** TASK-006

**Subtasks:**
- [ ] Install `python-magic` for content-based MIME type detection
- [ ] Validate file content (not just extension) on upload
- [ ] Reject files where detected MIME type does not match `audio/*`
- [ ] Return `UNSUPPORTED_FORMAT` with detected type in error message

---

#### TASK-027 · Add job resume capability
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-017

**Subtasks:**
- [ ] Persist job state to a JSON file at `/tmp/motif/{job_id}/job.json` after every stage
- [ ] On server restart, scan for incomplete jobs and resume from last completed stage
- [ ] Add `GET /api/v1/jobs` endpoint listing all jobs (for debugging)

---

#### TASK-028 · Improve subtitle styling and positioning
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 1 hour
- **Dependencies:** TASK-015

**Subtasks:**
- [ ] Add shadow effect to subtitles for readability on bright backgrounds
- [ ] Implement word highlighting: current word appears white, preceding words appear dim gray
- [ ] Ensure subtitles do not overlap the bottom 10% of the frame (safe zone)
- [ ] Test subtitle rendering with songs containing rapid-fire lyrics

---

#### TASK-029 · Add thumbnail generation
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 30 min
- **Dependencies:** TASK-018

**Subtasks:**
- [ ] After reel assembly, extract frame at 25% of total duration using FFmpeg
- [ ] Save as `thumbnail_{job_id}.jpg` (85% quality)
- [ ] Return `thumbnail_url` in job status response and on result screen

---

#### TASK-030 · Build advanced settings panel in UI
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 1 hour
- **Dependencies:** TASK-019

**Subtasks:**
- [ ] Add collapsible "Advanced Settings" accordion below duration selector
- [ ] LLM provider selector: Claude / Ollama with model name input for Ollama
- [ ] Image provider selector: Replicate / Local
- [ ] Seed input (numeric, default 42)
- [ ] Style preset selector (single option `cinematic` in V1, with hint "More styles in V2")

---

#### TASK-031 · Add energy-reactive cut pacing
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 1 hour
- **Dependencies:** TASK-016

**Subtasks:**
- [ ] Implement `compute_cut_duration(scene, energy_at_beat) -> int`
  - High energy (>0.8): multiply scene duration by 0.7 (faster cuts)
  - Low energy (<0.4): multiply scene duration by 1.3 (slower cuts)
  - Clamp to min 1500ms, max 8000ms
- [ ] Apply to scene timing during FFmpeg assembly
- [ ] Add `energy_pacing_enabled` config option (default: true)

---

#### TASK-032 · Add per-scene regeneration API endpoint
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 1.5 hours
- **Dependencies:** TASK-017

**Subtasks:**
- [ ] Implement `POST /api/v1/jobs/:id/scenes/:sid/regenerate`
  - Delete existing scene image
  - Increment seed by 100 (avoid same image)
  - Call `sge.generate_image()` for that scene only
  - Re-run `cse.assemble_reel()` with updated scene
  - Return updated job status
- [ ] Add "Regenerate" button to each `<SceneCard>` in the result UI

---

#### TASK-033 · Improve error messages in UI
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 45 min
- **Dependencies:** TASK-021

**Subtasks:**
- [ ] Create `frontend/lib/errors.ts` — maps error codes to user-facing messages + actions
- [ ] Display error banner with code, message, and suggested action on processing screen
- [ ] Add "Try Again" button that clears the failed job and returns to upload screen
- [ ] Add FAQ accordion at bottom of upload screen for common issues

---

#### TASK-034 · Write unit tests for pipeline modules
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 3 hours
- **Dependencies:** TASK-022

**Subtasks:**
- [ ] Create `backend/tests/` directory with `pytest` setup
- [ ] `test_apm.py`: test `detect_beats()` with a reference audio file; assert BPM within ±5 of known value
- [ ] `test_lie.py`: test JSON parsing fallback strategies with 5 malformed JSON strings
- [ ] `test_lie.py`: test `validate_scene_plan()` with edge cases (float timestamps, missing fields)
- [ ] `test_cse.py`: test `compute_beat_snapped_cuts()` with a synthetic beat grid
- [ ] `test_cse.py`: test subtitle ASS file generation with sample lyrics
- [ ] Run tests in CI via `pytest backend/tests/`

---

#### TASK-035 · Set up GitHub Actions CI
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 1 hour
- **Dependencies:** TASK-034

**Subtasks:**
- [ ] Create `.github/workflows/ci.yml`
  - Trigger: push to `main` and pull requests
  - Jobs: lint Python (ruff), run pytest, lint TypeScript (eslint), build Next.js
- [ ] Add `pytest` badge to README
- [ ] Add build status badge to README

---

#### TASK-036 · Write contributing guide
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 45 min
- **Dependencies:** TASK-024

**Subtasks:**
- [ ] Create `CONTRIBUTING.md`
  - Local development setup (without Docker)
  - How to add a new style preset
  - How to add a new LLM provider
  - How to run tests
  - PR process and branch naming convention
- [ ] Create `docs/architecture.md` — prose explanation of the pipeline with diagrams

---

#### TASK-037 · Performance profiling and optimization
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 2 hours
- **Dependencies:** TASK-022

**Subtasks:**
- [ ] Profile a full 60s reel run and identify the 3 slowest operations
- [ ] If SDXL loading time > 10s: verify singleton model loading is working correctly
- [ ] If FFmpeg composition > 60s: explore GPU-accelerated encoding (`-hwaccel cuda`)
- [ ] If Whisper > 60s: test `small` vs `base` quality/speed tradeoff
- [ ] Document profiling results in `docs/performance.md`

---

#### TASK-038 · Release V1.0 on GitHub
- **Status:** `[ ]`
- **Priority:** P1
- **Estimate:** 30 min
- **Dependencies:** TASK-037

**Subtasks:**
- [ ] Create GitHub Release `v1.0.0` with changelog
- [ ] Write release notes: what's included, known limitations, what's coming in V2
- [ ] Update README version badge
- [ ] Post on relevant communities (optional: Hacker News Show HN, Reddit r/MachineLearning)

---

## Phase 3 — V2 Features
> **Goal:** Expand into the full Motif V2 architecture described in DESIGN_v2.md

---

#### TASK-039 · Style Preset Engine — 6 presets
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 3 days
- **Dependencies:** TASK-038

**Subtasks:**
- [ ] Design and implement `StylePresetEngine` class with 6 presets
- [ ] Write prompt prefix/suffix + color rules for each preset: cinematic, anime, illustration, pixel_art, documentary, abstract
- [ ] Implement model routing per preset (RunwayML for cinematic/documentary; SDXL anime checkpoint for anime)
- [ ] Add style preview thumbnails to UI upload screen
- [ ] Add per-preset subtitle font and transition style

---

#### TASK-040 · Character Consistency Module
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 4 days
- **Dependencies:** TASK-039

**Subtasks:**
- [ ] Implement character extraction in LIE: parse character names and descriptions from narrative
- [ ] Implement `CharacterConsistencyModule` class
- [ ] Generate 4-angle reference images per character using SDXL before main scene generation
- [ ] Integrate IP-Adapter Plus for character injection into scene generation calls
- [ ] Implement per-scene injection weight based on shot type (see requirements.md §4 CCM table)
- [ ] Add character cards to processing screen UI

---

#### TASK-041 · Multilingual pipeline
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 2 days
- **Dependencies:** TASK-038

**Subtasks:**
- [ ] Switch Whisper to `large-v3` multilingual checkpoint (or make configurable)
- [ ] Add romanization pass for CJK and Arabic scripts
- [ ] Update LIE prompts to be language-aware (pass detected language to LLM)
- [ ] Add Cultural Context Adapter to LIE system prompt
- [ ] Add subtitle font selection per language (Noto Sans family)
- [ ] Add RTL subtitle handling for Arabic/Hebrew
- [ ] Test with Korean, Spanish, Hindi, and Arabic songs

---

#### TASK-042 · Manual Storyboard Editor
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 5 days
- **Dependencies:** TASK-040

**Subtasks:**
- [ ] Design storyboard editor data model and API endpoints (see requirements.md FR-API-01 V2 additions)
- [ ] Implement storyboard CRUD endpoints (`GET/PUT/PATCH/DELETE /api/v2/jobs/:id/storyboard/...`)
- [ ] Build `<StoryboardEditor>` React component with drag-and-drop (use `@dnd-kit/sortable`)
- [ ] Build `<SceneEditorPanel>` right-side panel for editing individual scene properties
- [ ] Implement autosave to server every 10 seconds
- [ ] Add "Approve & Generate" CTA that locks storyboard and triggers SGE

---

#### TASK-043 · Batch parallel export
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 2 days
- **Dependencies:** TASK-038

**Subtasks:**
- [ ] Implement `batch_export(job: Job, platforms: list) -> dict`
  - Run 4 FFmpeg encode workers in parallel using `asyncio.gather`
  - Platform specs: Instagram (H.264 30fps 90s), TikTok (H.264 30fps 60s), YouTube (H.264 60fps 60s), Universal
- [ ] Implement `POST /api/v2/jobs/:id/export/batch` endpoint
- [ ] Implement `GET /api/v2/jobs/:id/export/batch/status` with per-platform progress
- [ ] Build `<BatchExportStatus>` UI component with 4 parallel progress bars
- [ ] Implement H.265 master archive export and 30-day retention

---

#### TASK-044 · RunwayML Gen-3 integration
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 2 days
- **Dependencies:** TASK-039

**Subtasks:**
- [ ] Sign up for RunwayML API access
- [ ] Implement `generate_video_runwayml(prompt, seed_image, duration, scene_id) -> str`
- [ ] Add `runwayml` as option for `IMAGE_PROVIDER` env var
- [ ] Implement key scene budget: key scenes use 8s video clips, others use 4s or SDXL stills
- [ ] Update CSE to handle MP4 scene clips instead of PNG images (remove Ken Burns for video inputs)

---

#### TASK-045 · Redis job queue
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 1.5 days
- **Dependencies:** TASK-038

**Subtasks:**
- [ ] Add Redis service to `docker-compose.yml`
- [ ] Replace in-memory job dict with Redis-backed store
- [ ] Implement job queue using BullMQ (Node) or `arq` (Python)
- [ ] Support multiple concurrent jobs (up to 3)
- [ ] Add queue position display to processing screen

---

#### TASK-046 · User authentication
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 3 days
- **Dependencies:** TASK-045

**Subtasks:**
- [ ] Add user model and database (SQLite for dev, PostgreSQL for prod)
- [ ] Implement email + password registration and login using `fastapi-users`
- [ ] Implement JWT session tokens
- [ ] Protect all job endpoints: users can only access their own jobs
- [ ] Add login / register screens to frontend

---

#### TASK-047 · Job history screen
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 1 day
- **Dependencies:** TASK-046

**Subtasks:**
- [ ] Implement `GET /api/v2/users/me/jobs` — paginated list of user's jobs
- [ ] Build `<JobHistoryScreen>` — grid of past reels with thumbnails, song name, date, status
- [ ] Add navigation link to history from the result screen

---

#### TASK-048 · Rate limiting and abuse prevention
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 1 day
- **Dependencies:** TASK-046

**Subtasks:**
- [ ] Add `slowapi` rate limiting middleware to FastAPI
- [ ] Limit: 5 job creations per user per hour (configurable)
- [ ] Limit: 100 API requests per IP per minute (unauthenticated)
- [ ] Add max file size enforcement at the reverse proxy level (nginx config)
- [ ] Return `RATE_LIMIT_EXCEEDED` error with Retry-After header

---

#### TASK-049 · Monitoring and observability
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 1.5 days
- **Dependencies:** TASK-045

**Subtasks:**
- [ ] Add Prometheus metrics endpoint `GET /metrics`
  - Track: jobs created per hour, jobs completed, jobs failed, avg generation time per stage
- [ ] Add OpenTelemetry tracing — trace each job across all pipeline stages
- [ ] Configure Grafana dashboard with key charts
- [ ] Add alerting: alert if job failure rate > 10% in any 15-minute window

---

#### TASK-050 · Cloud deployment guide
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 1 day
- **Dependencies:** TASK-038

**Subtasks:**
- [ ] Write `docs/deployment.md` — step-by-step guide to deploy Motif to AWS
  - EC2 instance types for backend (g5.2xlarge) and frontend (t3.medium)
  - S3 bucket for output file storage
  - CloudFront CDN for download delivery
  - Environment variable setup on EC2
  - nginx reverse proxy configuration
- [ ] Create Terraform config for infrastructure provisioning (optional)
- [ ] Document estimated monthly AWS cost at different usage levels (100 / 1000 / 10000 reels/month)

---

#### TASK-051 · V2 release
- **Status:** `[ ]`
- **Priority:** P2
- **Estimate:** 1 day
- **Dependencies:** All Phase 3 tasks

**Subtasks:**
- [ ] Full regression test of V2 feature set
- [ ] Update `README.md`, `requirements.md`, `task.md` to reflect V2 state
- [ ] Create GitHub Release `v2.0.0` with full changelog
- [ ] Update `DESIGN_v2.md` with any implementation deviations

---

## Appendix

### A. Task Priority Legend

| Priority | Label | Meaning |
|----------|-------|---------|
| P0 | Must ship | Prototype is broken without it |
| P1 | Should ship | V1 is not polished without it |
| P2 | Could ship | V2 feature — deferred from prototype |

### B. Effort Estimate Guide

| Size | Range | Examples |
|------|-------|---------|
| XS | < 30 min | Update a config, add a field, fix a bug |
| S | 30–60 min | One module function, one UI component |
| M | 1–2 hours | One pipeline stage, one screen |
| L | 2–4 hours | Full module, integration work |
| XL | 1+ days | New subsystem, complex integration |

### C. Branching Strategy

```
main            ← stable releases only
  └── develop   ← integration branch
        ├── feature/TASK-008-whisper-transcription
        ├── feature/TASK-010-claude-scene-planner
        └── fix/TASK-016-ffmpeg-zoompan-filter
```

**Commit format:** `type(scope): description`  
Types: `feat`, `fix`, `chore`, `docs`, `test`, `refactor`, `perf`

### D. Definition of Done (Global)

A task is `[x]` Done when ALL of the following are true:
- Code is written and locally tested
- No linting errors (`ruff` for Python, `eslint` for TypeScript)
- Unit tests written where applicable and passing
- Code pushed to feature branch and PR opened
- PR description links to Task ID and describes what was done
- PR reviewed and merged to `develop`

---

*End of task.md — Motif v1.0*  
*Total tracked tasks: 51 across 3 phases*
