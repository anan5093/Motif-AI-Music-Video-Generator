# requirements.md — Motif
**Project:** Motif — AI Music Video Generator  
**Version:** 1.0  
**Status:** Pre-development  
**Last Updated:** April 2026  
**Author:** Senior AI Software Designer  

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Stakeholders](#2-stakeholders)
3. [System Context](#3-system-context)
4. [Functional Requirements](#4-functional-requirements)
   - 4.1 Audio Upload & Processing
   - 4.2 Lyrics Intelligence
   - 4.3 Style & Visual Configuration
   - 4.4 Scene Generation
   - 4.5 Video Composition & Sync
   - 4.6 Export & Delivery
   - 4.7 User Interface
   - 4.8 API
5. [Non-Functional Requirements](#5-non-functional-requirements)
   - 5.1 Performance
   - 5.2 Reliability & Availability
   - 5.3 Security
   - 5.4 Scalability
   - 5.5 Usability
   - 5.6 Maintainability
6. [Data Requirements](#6-data-requirements)
7. [Integration Requirements](#7-integration-requirements)
8. [Constraints & Assumptions](#8-constraints--assumptions)
9. [Acceptance Criteria](#9-acceptance-criteria)
10. [Glossary](#10-glossary)

---

## 1. Project Overview

### 1.1 Purpose

Motif is an AI-powered short-form video generation platform. Given any song as input, Motif automatically produces a narrative-driven, beat-synchronized video reel (30–90 seconds) suitable for Instagram Reels, TikTok, and YouTube Shorts.

The system reads the song's lyrics, extracts the story and emotional arc, generates matching visual scenes using generative AI, assembles all scenes with beat-snapped cuts, overlays lyric subtitles, and mixes the original song as the audio track.

### 1.2 Problem Statement

Creators, musicians, and marketers who want to produce short-form video content from their music face two blockers:

- **Skill gap** — video editing, storyboarding, and motion design require specialized expertise most creators don't have.
- **Time and cost** — professional music video production is expensive; even basic lyric videos take hours of manual work.

Motif removes both blockers by automating the entire creative pipeline from audio input to finished reel output.

### 1.3 Product Goals

| Goal ID | Goal | Priority |
|---------|------|----------|
| G-01 | Produce a complete reel from a song file with zero manual editing | Must Have |
| G-02 | Reel visuals must accurately reflect lyrical narrative, not just mood | Must Have |
| G-03 | Video cuts and transitions must be synchronized to the song's beat | Must Have |
| G-04 | Output files must meet each platform's technical specifications | Must Have |
| G-05 | End-to-end generation must complete within 15 minutes for a 60s reel | Should Have |
| G-06 | Support multiple visual style presets per reel | Should Have |
| G-07 | Support multilingual songs (30+ languages) | Should Have |
| G-08 | Allow manual storyboard editing before generation begins | Could Have |
| G-09 | Run fully locally with no paid API dependencies | Could Have |

### 1.4 Scope

**In scope (V1 Prototype):**
- Song upload (MP3, WAV, M4A, FLAC, OGG)
- Whisper-based local transcription with word-level timestamps
- Claude / Ollama LLM scene planning from lyrics
- SDXL / Replicate image generation per scene
- FFmpeg Ken Burns animation + audio composition
- Lyric subtitle overlay
- Single MP4 reel output (1080×1920, 9:16)
- Basic web UI (upload → progress → preview → download)

**Out of scope (V1):**
- Real-time video generation (no RunwayML in prototype)
- Character consistency module
- Style preset engine
- Batch platform export
- User accounts, authentication
- Payment / subscription
- Mobile application

---

## 2. Stakeholders

| Role | Name / Team | Responsibility | Interest |
|------|-------------|----------------|----------|
| Product Owner | Project Lead | Define requirements, prioritize backlog | High — all decisions |
| Lead Developer | Backend Engineer | Pipeline implementation, API design | High — technical delivery |
| Frontend Developer | UI Engineer | Web interface, real-time progress | High — user experience |
| AI/ML Engineer | Generative AI | LLM prompt design, image gen integration | High — output quality |
| End Users | Creators, musicians, marketers | Upload songs, download reels | High — value delivery |
| DevOps | Infrastructure | Deployment, Docker, CI/CD | Medium |

---

## 3. System Context

### 3.1 Context Diagram

```
                        ┌─────────────────────────────────┐
                        │            MOTIF                │
                        │                                 │
  [User / Browser] ────►│  Frontend (Next.js)             │
                        │       │                         │
                        │  Backend (FastAPI)              │
                        │       │                         │
                        │  Pipeline Workers               │
                        │  APM │ LIE │ SGE │ CSE │ EDM    │
                        └──┬───┴──┬──┴──┬──┴────┴────────┘
                           │      │     │
                    Whisper│  Ollama   Replicate/
                   librosa │  (local)  local SDXL
                   (local) │           FFmpeg (local)
```

### 3.2 External Interfaces

| Interface | Type | Direction | Protocol |
|-----------|------|-----------|----------|
| Anthropic Claude API | AI | Outbound | HTTPS REST |
| Ollama (local) | AI | Local | HTTP REST (localhost:11434) |
| Replicate API | AI | Outbound | HTTPS REST |
| HuggingFace Hub | Model weights | Outbound (one-time) | HTTPS |
| FFmpeg | CLI tool | Local | Subprocess |
| Browser client | UI | Inbound | HTTP / WebSocket |

---

## 4. Functional Requirements

> **Notation:**  
> `MUST` — mandatory for V1 release  
> `SHOULD` — important, included if time allows  
> `COULD` — desirable, deferred to V2  

---

### 4.1 Audio Upload & Processing (APM)

#### FR-APM-01 — File Upload
The system **MUST** accept audio file uploads via a web UI drag-and-drop zone and via a REST API endpoint (`POST /api/v1/jobs`).

**Accepted formats:** MP3, WAV, M4A, FLAC, OGG  
**Maximum file size:** 50MB  
**Maximum audio duration:** 10 minutes (processing pipeline selects a 30–90s window)

#### FR-APM-02 — File Validation
The system **MUST** validate uploaded files before processing:
- Reject files larger than 50MB with error code `FILE_TOO_LARGE`
- Reject unsupported formats with error code `UNSUPPORTED_FORMAT`
- Reject silent or corrupt audio files with error code `INVALID_AUDIO`

#### FR-APM-03 — Vocal Transcription
The system **MUST** transcribe song lyrics from the audio using OpenAI Whisper.

**Requirements:**
- Word-level timestamps must be produced for every transcribed word
- Transcription must return segment-level groupings (lines)
- The model used must be configurable via environment variable (`WHISPER_MODEL`: base / small / medium / large-v3)
- Default model for prototype: `base`

#### FR-APM-04 — Language Detection
The system **MUST** detect the language of the lyrics automatically from the Whisper output language token.

The detected language **MUST** be stored in the job record and passed to the Lyrics Intelligence Engine.

#### FR-APM-05 — Beat Detection
The system **MUST** extract tempo (BPM) and a beat grid (array of beat timestamps in milliseconds) using `librosa`.

**Output must include:**
- `bpm` — tempo in beats per minute (float, rounded to nearest integer)
- `beat_grid_ms` — list of all beat positions in milliseconds
- `energy_curve` — normalized energy level per 500ms window (0.0–1.0)

#### FR-APM-06 — Section Segmentation
The system **SHOULD** segment the song into structural sections (intro / verse / chorus / bridge / outro) using energy analysis.

If section detection fails, the system **MUST** fall back to uniform time windows (every 8 seconds = one section).

#### FR-APM-07 — Instrumental Detection
The system **SHOULD** detect songs with no lyrics (instrumentals) and route them to energy-only visual mode, which generates abstract visuals driven by the energy curve and beat grid rather than lyric narrative.

---

### 4.2 Lyrics Intelligence Engine (LIE)

#### FR-LIE-01 — Scene Plan Generation
The system **MUST** call an LLM (Claude API or local Ollama model) to analyze transcribed lyrics and produce a structured scene plan.

**Scene plan must include for each scene:**
- `scene_id` — unique identifier (e.g., `s001`)
- `start_ms` — scene start time in milliseconds
- `end_ms` — scene end time in milliseconds
- `lyric_lines` — list of lyric lines this scene illustrates
- `t2v_prompt` — text-to-image/video prompt (max 100 words)
- `mood` — dominant emotional tone
- `shot_type` — camera framing (wide / medium / close-up / aerial / macro)

#### FR-LIE-02 — Narrative Extraction
The scene plan **MUST** include a top-level `narrative` field: a one-to-two sentence summary of the song's story or emotional theme.

The system **MUST** extract the story (not just mood or genre) and reflect it in the visual prompts. A song about leaving home must generate scenes of departure — not generic "sad" visuals.

#### FR-LIE-03 — Scene Count & Duration
The system **MUST** generate between 6 and 12 scenes for a 60-second reel, proportionally adjusted for 30s or 90s targets.

All scene `end_ms` timestamps must sum to within ±3 seconds of the target reel duration.

#### FR-LIE-04 — Timestamp Alignment
Every scene's `start_ms` **MUST** align with a lyric phrase boundary (not mid-word or mid-sentence). The system **SHOULD** snap scene start times to the nearest beat grid position within ±300ms.

#### FR-LIE-05 — LLM Provider Switching
The system **MUST** support two LLM provider modes, switchable via environment variable (`LLM_PROVIDER`):

| Value | Provider | Model |
|-------|---------|-------|
| `claude` | Anthropic API | claude-sonnet-4-6 |
| `ollama` | Local Ollama | configurable via `OLLAMA_MODEL` |

The pipeline code must not change when switching providers — only the environment variable changes.

#### FR-LIE-06 — JSON Reliability & Retry
The system **MUST** handle malformed LLM JSON output without crashing.

**Recovery strategy:**
1. Attempt direct JSON parse
2. Strip markdown fences and retry parse
3. Extract outermost `{...}` block and retry parse
4. If all three fail, retry the full LLM call once with a stricter prompt
5. If retry fails, return error `LIE_PARSE_FAILED` to the job and surface it to the user

#### FR-LIE-07 — Cultural Context
The system **SHOULD** instruct the LLM to generate culturally appropriate scene descriptions based on the detected song language. A Korean song should produce scenes reflecting Korean aesthetics by default unless lyrics explicitly describe a different setting.

---

### 4.3 Style & Visual Configuration

#### FR-STY-01 — Style Preset Selection
The system **SHOULD** support at least one configurable visual style preset for V1 (prototype uses `cinematic` only).

**V1 preset:** `cinematic` — film grain, golden hour lighting, shallow depth of field, desaturated color grade.

**V2 presets (planned):** `anime`, `illustration`, `pixel_art`, `documentary`, `abstract`.

#### FR-STY-02 — Style Prompt Injection
The selected style preset **MUST** inject a prefix and suffix into every `t2v_prompt` before sending to the image generation model.

**Example for `cinematic`:**
```
PREFIX: "Cinematic film still, golden hour lighting, 35mm film grain, "
SUFFIX: ", shallow depth of field, color graded, ultra high quality"
```

#### FR-STY-03 — Negative Prompt
Every image generation call **MUST** include a negative prompt to prevent common quality issues.

**Default negative prompt:**
`"text, watermark, blurry, cartoon, anime, low quality, ugly, deformed, oversaturated"`

#### FR-STY-04 — Duration Selection
The system **MUST** allow the user to select the target reel duration before processing:
- 30 seconds
- 45 seconds
- 60 seconds (default)
- 90 seconds

---

### 4.4 Scene Generation Engine (SGE)

#### FR-SGE-01 — Image Generation Provider
The system **MUST** support two image generation modes, switchable via environment variable (`IMAGE_PROVIDER`):

| Value | Provider | Model |
|-------|---------|-------|
| `replicate` | Replicate API | SDXL 1.0 |
| `local` | Local diffusers | SDXL 1.0 or SD-Turbo (CPU mode) |

#### FR-SGE-02 — Image Resolution
All generated images **MUST** be in portrait 9:16 aspect ratio.

**Target dimensions:**
- GPU mode: 1024 × 1792 pixels
- CPU / SD-Turbo mode: 576 × 1024 pixels (upscaled by FFmpeg during composition)

#### FR-SGE-03 — Scene Image Output
The system **MUST** save each generated scene image as a PNG file to the configured output directory with filename pattern `scene_{scene_id}.png`.

#### FR-SGE-04 — Skip on Existing File
The system **MUST** skip image generation for any scene where `scene_{scene_id}.png` already exists in the output directory. This allows interrupted jobs to resume without regenerating completed scenes.

#### FR-SGE-05 — Generation Seed
Each scene **MUST** use a deterministic seed (`base_seed + scene_index`) so that regenerating the same job produces identical results unless the seed is changed.

The seed **SHOULD** be configurable per job via the API.

#### FR-SGE-06 — Sequential Generation
For V1, scenes **MUST** be generated sequentially (one at a time). Parallel generation is deferred to V2.

After each scene completes, the system **MUST** emit a WebSocket event (`scene_ready`) with the scene ID and image URL so the UI can update in real time.

#### FR-SGE-07 — CPU Mode
When `CPU_MODE=true` is set in environment, the system **MUST** use `sd-turbo` with 4 inference steps and no CFG guidance. This mode sacrifices image quality for hardware compatibility.

---

### 4.5 Video Composition & Sync Engine (CSE)

#### FR-CSE-01 — Ken Burns Animation
The system **MUST** apply a slow pan/zoom (Ken Burns effect) to each scene image to create motion from static images.

**Default animation:** slow zoom from 1.0× to 1.05× over the scene duration, combined with a subtle pan in a randomized direction (left, right, or center).

#### FR-CSE-02 — Beat-Snapped Cuts
Every cut between scenes **MUST** be snapped to the nearest beat in the beat grid within a ±300ms tolerance window.

If no beat falls within tolerance, the cut is placed at the scene's raw `start_ms` timestamp.

#### FR-CSE-03 — Lyric Subtitle Overlay
The system **MUST** overlay lyric subtitles on the video.

**Requirements:**
- Subtitles appear word-by-word (karaoke style) synchronized to Whisper word timestamps
- Subtitles are positioned at the bottom third of the frame (below 65% vertical)
- Font: Helvetica Neue or system fallback sans-serif
- Font size: 52px
- Color: white with black outline (2px stroke) for readability on any background
- The subtitle layer must be generated as an ASS subtitle file and burned in via FFmpeg

#### FR-CSE-04 — Audio Mixing
The system **MUST** use the original uploaded song as the audio track for the output video.

Any ambient sound from image generation (none expected) **MUST** be ducked or muted. The song audio level must not be altered.

#### FR-CSE-05 — Output Format
The system **MUST** produce a single MP4 file with:
- Video codec: H.264 (libx264)
- Video CRF: 18 (high quality)
- Frame rate: 30fps
- Resolution: 1080 × 1920
- Audio codec: AAC
- Audio bitrate: 192 kbps
- Container: MP4

#### FR-CSE-06 — Duration Enforcement
The output video **MUST** be trimmed to exactly the user-selected duration. If composed video is longer, it must be cut. If shorter (due to timing rounding), the last frame must be held and faded to black.

---

### 4.6 Export & Delivery

#### FR-EXP-01 — File Download
The system **MUST** provide a download endpoint (`GET /api/v1/jobs/:id/download`) that returns the final MP4 file.

#### FR-EXP-02 — Preview URL
After reel generation completes, the system **MUST** provide a streaming preview URL that the frontend video player can load without downloading the full file.

#### FR-EXP-03 — Thumbnail Generation
The system **SHOULD** automatically extract a thumbnail from the most visually prominent frame (first frame of the longest scene) and store it as `thumbnail_{job_id}.jpg`.

#### FR-EXP-04 — File Retention
Generated reels **MUST** be retained for a minimum of 24 hours after generation. Files older than 24 hours may be deleted by a scheduled cleanup task.

Source audio files **MUST** be deleted immediately after the APM stage completes (not retained in full for privacy).

---

### 4.7 User Interface

#### FR-UI-01 — Upload Screen
The upload screen **MUST** include:
- Audio file drag-and-drop zone (with click-to-browse fallback)
- Visual feedback on drag hover and during file reading
- Duration selector (30s / 45s / 60s / 90s) with 60s pre-selected
- LLM provider selector (Claude / Ollama) — collapsible advanced option
- Submit / Generate button (disabled until a valid file is selected)

#### FR-UI-02 — Processing Screen
The processing screen **MUST** display:
- Job ID for reference
- A 5-stage progress indicator with labels: Transcription → Scene Planning → Image Generation → Composition → Ready
- Estimated time remaining (updated every 10 seconds)
- Live scene cards that appear as each scene image completes (via WebSocket)

#### FR-UI-03 — Scene Preview Cards
As each scene image is generated, a card **MUST** appear in the UI showing:
- The generated scene image (thumbnail)
- The lyric line(s) the scene illustrates
- The scene mood tag
- Scene duration in seconds

#### FR-UI-04 — Result Screen
The result screen **MUST** include:
- A 9:16 inline video player with play/pause/seek controls
- A download button (triggers `GET /api/v1/jobs/:id/download`)
- A "Generate Another" button that returns to the upload screen
- Display of total generation time

#### FR-UI-05 — Error States
The UI **MUST** display user-friendly error messages for all error codes returned by the API. Error messages must suggest a recovery action.

**Example:**
- `FILE_TOO_LARGE` → "Your file is over 50MB. Try exporting at a lower bitrate."
- `LIE_PARSE_FAILED` → "Scene planning failed. Try switching to Claude in advanced settings."
- `OLLAMA_UNAVAILABLE` → "Ollama is not running. Start it with: ollama serve"

#### FR-UI-06 — Responsive Layout
The UI **MUST** be functional on desktop browsers (Chrome, Firefox, Safari, Edge — latest two versions). Mobile browser support is desirable but not required for V1.

---

### 4.8 API

#### FR-API-01 — Job Creation
```
POST /api/v1/jobs
Content-Type: multipart/form-data

Body:
  file         (required) Audio file binary
  duration_s   (optional, default: 60) Target reel duration in seconds: 30|45|60|90
  style        (optional, default: "cinematic") Style preset name
  llm_provider (optional, default: env variable) "claude" | "ollama"
  seed         (optional, default: 42) Integer seed for reproducibility

Response 202:
{
  "job_id": "j_abc123",
  "status": "queued",
  "estimated_seconds": 180,
  "created_at": "2026-04-26T10:00:00Z"
}
```

#### FR-API-02 — Job Status
```
GET /api/v1/jobs/:id

Response 200:
{
  "job_id": "j_abc123",
  "status": "processing" | "complete" | "failed",
  "stage": "apm" | "lie" | "sge" | "cse" | "done",
  "progress": 0-100,
  "error": null | { "code": string, "message": string },
  "scenes": [ ... ] (populated after LIE stage),
  "result": null | { "preview_url": string, "download_url": string, "thumbnail_url": string }
}
```

#### FR-API-03 — File Download
```
GET /api/v1/jobs/:id/download

Response 200:
Content-Type: video/mp4
Content-Disposition: attachment; filename="motif_{job_id}.mp4"
[binary MP4 file stream]

Response 404: { "error": "JOB_NOT_FOUND" }
Response 409: { "error": "JOB_NOT_COMPLETE" }
```

#### FR-API-04 — WebSocket Progress
```
WebSocket: ws://host/api/v1/jobs/:id/events

Server emits:
  { "type": "stage_update",  "stage": string, "progress": int, "message": string }
  { "type": "scene_ready",   "scene_id": string, "image_url": string, "lyric_lines": [] }
  { "type": "reel_ready",    "preview_url": string, "download_url": string }
  { "type": "error",         "code": string, "message": string, "recoverable": bool }
```

#### FR-API-05 — Health Check
```
GET /api/v1/health

Response 200:
{
  "status": "ok",
  "whisper_model": "base",
  "llm_provider": "ollama",
  "ollama_available": true,
  "image_provider": "local",
  "gpu_available": true,
  "ffmpeg_version": "6.1.1"
}
```

---

## 5. Non-Functional Requirements

### 5.1 Performance

| ID | Requirement | Target | Priority |
|----|-------------|--------|----------|
| NFR-PERF-01 | End-to-end generation time for 60s reel (GPU) | ≤ 10 minutes | Must |
| NFR-PERF-02 | End-to-end generation time for 60s reel (CPU) | ≤ 45 minutes | Should |
| NFR-PERF-03 | API response time for non-pipeline endpoints | ≤ 300ms | Must |
| NFR-PERF-04 | File upload to job start latency | ≤ 5 seconds | Must |
| NFR-PERF-05 | First scene image delivered to UI | ≤ 3 minutes after job start | Should |
| NFR-PERF-06 | FFmpeg composition time for 60s reel | ≤ 90 seconds | Must |

### 5.2 Reliability & Availability

| ID | Requirement |
|----|-------------|
| NFR-REL-01 | Each pipeline stage must have at least one automatic retry on transient failure before marking the job as failed |
| NFR-REL-02 | Jobs that fail mid-pipeline must preserve completed stage outputs so that re-runs can skip completed stages |
| NFR-REL-03 | The system must handle Ollama unavailability gracefully — surface a clear error rather than hanging indefinitely |
| NFR-REL-04 | The system must handle Replicate API timeouts with a fallback to local image generation if `local` mode is available |
| NFR-REL-05 | The API server must return a 503 with a meaningful message if a required dependency (FFmpeg, Whisper) is not installed |

### 5.3 Security

| ID | Requirement |
|----|-------------|
| NFR-SEC-01 | All API keys (Anthropic, Replicate) must be stored in environment variables only — never hardcoded in source code or committed to the repository |
| NFR-SEC-02 | Uploaded audio files must be stored with a randomly generated filename (UUID-based) — never using the original filename directly to prevent path traversal |
| NFR-SEC-03 | File type validation must be performed on file content (MIME type inspection), not just the file extension |
| NFR-SEC-04 | All generated file paths must be validated to be within the configured output directory before reading or writing |
| NFR-SEC-05 | The API must validate and enforce the 50MB file size limit before reading the file into memory |
| NFR-SEC-06 | The `.env` file must be listed in `.gitignore`. The repository must only contain `.env.example` |
| NFR-SEC-07 | CORS policy must be configured to restrict API access to the configured frontend origin |

### 5.4 Scalability

| ID | Requirement |
|----|-------------|
| NFR-SCA-01 | The pipeline must be designed as independent, composable stages so that any stage can be replaced or scaled independently |
| NFR-SCA-02 | The system must process one job at a time in V1. Concurrent job support is a V2 requirement |
| NFR-SCA-03 | The image generation model must be loaded once and reused across all scenes within a job — not reloaded per scene |
| NFR-SCA-04 | Output directories must be partitioned by job ID to prevent filename collisions across jobs |

### 5.5 Usability

| ID | Requirement |
|----|-------------|
| NFR-USA-01 | A user with no technical background must be able to upload a song and download a reel with zero configuration required |
| NFR-USA-02 | The README must include a Quick Start section that gets a developer from clone to running first reel in under 30 minutes |
| NFR-USA-03 | All error messages shown in the UI must be in plain English and include a suggested action |
| NFR-USA-04 | The processing screen must show estimated time remaining so users know whether to wait or come back later |
| NFR-USA-05 | The video preview must auto-play muted on the result screen and display controls for the user to unmute |

### 5.6 Maintainability

| ID | Requirement |
|----|-------------|
| NFR-MNT-01 | Each pipeline stage must be a separate Python module (file) with a single public function — no cross-stage imports |
| NFR-MNT-02 | All configuration values must be loaded from environment variables via a single `config.py` file — not scattered across modules |
| NFR-MNT-03 | All inter-stage data must conform to Pydantic models defined in `models.py` |
| NFR-MNT-04 | The project must include a `docker-compose.yml` that starts the full system with a single command |
| NFR-MNT-05 | Each module must include docstrings explaining its inputs, outputs, and any external dependencies |

---

## 6. Data Requirements

### 6.1 Job Record Schema

```python
class Job(BaseModel):
    job_id: str                     # UUID, e.g. "j_a1b2c3d4"
    status: JobStatus               # queued | processing | complete | failed
    stage: PipelineStage            # apm | lie | sge | cse | done
    progress: int                   # 0-100
    created_at: datetime
    updated_at: datetime
    config: JobConfig               # user-supplied parameters
    apm_output: Optional[APMOutput] # populated after APM stage
    lie_output: Optional[LIEOutput] # populated after LIE stage
    sge_output: Optional[SGEOutput] # populated after SGE stage
    result: Optional[JobResult]     # populated after CSE stage
    error: Optional[JobError]       # populated on failure
```

### 6.2 APM Output Schema

```python
class APMOutput(BaseModel):
    lyrics: List[LyricWord]         # word-level with timestamps
    sections: List[SongSection]     # intro/verse/chorus etc.
    bpm: int
    beat_grid_ms: List[int]
    energy_curve: List[float]
    language: str                   # ISO 639-1 code, e.g. "en"
    language_confidence: float      # 0.0-1.0
    has_vocals: bool
    duration_ms: int
```

### 6.3 LIE Output Schema

```python
class LIEOutput(BaseModel):
    narrative: str
    emotional_arc: List[str]
    scenes: List[Scene]

class Scene(BaseModel):
    scene_id: str
    start_ms: int
    end_ms: int
    lyric_lines: List[str]
    t2v_prompt: str
    mood: str
    shot_type: str
    image_path: Optional[str]       # populated after SGE
    generation_status: str          # pending | complete | failed
```

### 6.4 File Storage Structure

```
/tmp/motif/
├── {job_id}/
│   ├── audio_original.{ext}       # deleted after APM
│   ├── audio_vocals.wav           # deleted after APM
│   ├── scene_s001.png
│   ├── scene_s002.png
│   ├── ...
│   ├── lyrics.ass                 # subtitle file
│   ├── scenes_list.txt            # FFmpeg concat manifest
│   ├── composed_no_audio.mp4      # intermediate
│   └── motif_{job_id}.mp4        # final output
```

### 6.5 Data Retention Policy

| Data Type | Retention | Deletion Trigger |
|-----------|-----------|-----------------|
| Uploaded audio | Until APM completes | APM stage finish |
| Scene images | 24 hours | Scheduled cleanup |
| Subtitle files | 24 hours | Scheduled cleanup |
| Intermediate MP4 | 24 hours | Scheduled cleanup |
| Final reel MP4 | 24 hours | Scheduled cleanup |
| Job metadata | 7 days | Scheduled cleanup |

---

## 7. Integration Requirements

### 7.1 Anthropic Claude API

| Requirement | Detail |
|-------------|--------|
| Endpoint | `https://api.anthropic.com/v1/messages` |
| Model | `claude-sonnet-4-6` |
| Auth | `ANTHROPIC_API_KEY` environment variable via `x-api-key` header |
| Timeout | 60 seconds |
| Retry | 1 automatic retry on 529 (overloaded) or 500 errors |
| Max tokens | 2048 per request |

### 7.2 Ollama (Local LLM)

| Requirement | Detail |
|-------------|--------|
| Base URL | `http://localhost:11434` (configurable via `OLLAMA_BASE_URL`) |
| Endpoint | `POST /api/generate` |
| Default model | `qwen2.5:7b` (configurable via `OLLAMA_MODEL`) |
| Timeout | 120 seconds |
| Availability check | `GET /api/tags` — called during health check |
| Temperature | 0.3 (fixed for JSON reliability) |

### 7.3 Replicate API

| Requirement | Detail |
|-------------|--------|
| Model | `stability-ai/sdxl:39ed52f2` |
| Auth | `REPLICATE_API_TOKEN` environment variable |
| Timeout | 120 seconds per image |
| Polling | Poll every 5 seconds for async predictions |
| Retry | 1 retry on timeout |

### 7.4 Local Diffusers (HuggingFace)

| Requirement | Detail |
|-------------|--------|
| GPU model | `stabilityai/stable-diffusion-xl-base-1.0` |
| CPU model | `stabilityai/sd-turbo` |
| Cache dir | `~/.cache/huggingface/` |
| First-run download | ~6.5GB (SDXL) or ~2.1GB (SD-Turbo) |
| Device | Auto-detected: `cuda` if available, else `cpu` |

### 7.5 FFmpeg

| Requirement | Detail |
|-------------|--------|
| Minimum version | 5.0 |
| Invocation | Subprocess call via `ffmpeg-python` wrapper |
| Required codecs | libx264, aac, subtitles filter |
| Installation check | Called during `/api/v1/health` endpoint |

---

## 8. Constraints & Assumptions

### 8.1 Technical Constraints

| ID | Constraint |
|----|-----------|
| CON-01 | FFmpeg must be installed on the host system — it cannot be bundled inside the Python package |
| CON-02 | Whisper `large-v3` model requires at least 10GB of VRAM; the prototype defaults to `base` which runs on CPU |
| CON-03 | SDXL image generation requires at least 6GB of VRAM for GPU mode; CPU mode using SD-Turbo works on any machine but is slow |
| CON-04 | Ollama must be running separately before the backend starts — it is not started by docker-compose |
| CON-05 | The system does not support Windows natively in V1; Docker is required for Windows users |
| CON-06 | The Replicate API does not support custom LoRA weights in the prototype integration |

### 8.2 Business Constraints

| ID | Constraint |
|----|-----------|
| CON-07 | The prototype must be completable by a solo developer within 3 days of focused work |
| CON-08 | Total API cost per demo run must not exceed $0.10 USD |
| CON-09 | The project must be fully open source (MIT license) |

### 8.3 Assumptions

| ID | Assumption |
|----|-----------|
| ASM-01 | Users are responsible for obtaining sync licenses for any songs they upload for commercial use |
| ASM-02 | The system will not be exposed to the public internet without adding rate limiting and auth (see V2 requirements) |
| ASM-03 | Songs submitted will primarily be in English for V1 — multilingual support is tested but not the primary focus |
| ASM-04 | The local Ollama model has already been pulled by the developer before running the system |
| ASM-05 | The developer machine has internet access for initial HuggingFace model weight downloads |

---

## 9. Acceptance Criteria

### 9.1 Pipeline Acceptance Tests

| Test ID | Test | Pass Condition |
|---------|------|----------------|
| AT-01 | Upload an English MP3 song (< 50MB) | Job created, `job_id` returned, status `queued` |
| AT-02 | Transcription completeness | Output contains word-level timestamps for ≥ 90% of audible lyrics |
| AT-03 | Scene plan validity | JSON parses cleanly, 6–12 scenes, all timestamps within song duration |
| AT-04 | Scene-lyric alignment | Each scene's `lyric_lines` contains text from the song (not hallucinated) |
| AT-05 | Image generation | One PNG file per scene exists in output directory after SGE stage |
| AT-06 | Beat sync | All scene cuts fall within ±300ms of a detected beat position |
| AT-07 | Output format | Final MP4 is 1080×1920, 30fps, H.264, AAC, within ±2s of target duration |
| AT-08 | Audio fidelity | Original song audio is present in the output MP4 without pitch or speed alteration |
| AT-09 | Subtitle sync | Subtitle words appear within ±200ms of their Whisper timestamps |
| AT-10 | Download endpoint | `GET /api/v1/jobs/:id/download` returns a valid, playable MP4 file |

### 9.2 Error Handling Acceptance Tests

| Test ID | Test | Pass Condition |
|---------|------|----------------|
| AT-11 | Upload a 60MB file | API returns 400 with `FILE_TOO_LARGE` before processing begins |
| AT-12 | Upload a `.txt` file renamed to `.mp3` | API returns 400 with `UNSUPPORTED_FORMAT` |
| AT-13 | Start Ollama job with Ollama not running | Job fails with `OLLAMA_UNAVAILABLE`, UI shows helpful error message |
| AT-14 | Ollama returns malformed JSON | System retries; if retry also fails, job fails with `LIE_PARSE_FAILED` |
| AT-15 | Poll a non-existent job ID | API returns 404 with `JOB_NOT_FOUND` |

### 9.3 Performance Acceptance Tests

| Test ID | Test | Pass Condition |
|---------|------|----------------|
| AT-16 | Generate 60s reel on GPU machine | Completes within 10 minutes |
| AT-17 | Generate 30s reel on CPU machine | Completes within 25 minutes |
| AT-18 | Health check endpoint | Responds within 500ms with accurate dependency status |

---

## 10. Glossary

| Term | Definition |
|------|-----------|
| **APM** | Audio Processing Module — the pipeline stage responsible for transcription, beat detection, and section analysis |
| **ASS** | Advanced SubStation Alpha — subtitle file format used by FFmpeg for styled subtitle burn-in |
| **Beat grid** | An array of timestamps (in milliseconds) representing every beat position in the song |
| **CSE** | Composition and Sync Engine — the FFmpeg assembly stage |
| **CRF** | Constant Rate Factor — FFmpeg video quality setting (lower = higher quality; 18 is near-lossless) |
| **Diffusers** | HuggingFace Python library for running local Stable Diffusion models |
| **Energy curve** | A normalized array (0.0–1.0) representing the audio energy level at each 500ms window |
| **IP-Adapter** | A technique for injecting reference image style into Stable Diffusion generations |
| **Ken Burns effect** | A slow pan and zoom animation applied to a still image to simulate camera movement |
| **LIE** | Lyrics Intelligence Engine — the LLM-based scene planning stage |
| **LoRA** | Low-Rank Adaptation — a lightweight fine-tuning technique for image generation models |
| **Motif** | The product name; also a musical/visual term for a recurring theme or idea |
| **Ollama** | A local LLM runtime that runs models like Llama 3, Qwen, and Mistral on a local machine |
| **Reel** | Short-form vertical video format (9:16, 30–90 seconds) used on Instagram, TikTok, YouTube Shorts |
| **SDXL** | Stable Diffusion XL — a high-quality open image generation model by Stability AI |
| **SD-Turbo** | A distilled, fast version of Stable Diffusion optimized for CPU and low-VRAM environments |
| **SGE** | Scene Generation Engine — the image generation stage |
| **Stem separation** | Isolating individual audio components (vocals, drums, bass) from a mixed track |
| **t2v_prompt** | Text-to-visual prompt — the text instruction sent to the image generation model for a scene |
| **Whisper** | OpenAI's open-source speech recognition model, used for lyric transcription |
| **Word timestamp** | The exact start and end time of a single spoken/sung word in an audio recording |

---

*End of requirements.md — Motif v1.0*
