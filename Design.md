# DESIGN.md — Motif-AI-Music-Video-Generator
**Version:** 2.0  
**Author:** Anand Raj  
**Last Updated:** April 2026
Previous Version: 1.0
**Status:** Active Development

---

## Changelog: V1 → V2

| Area | V1 | V2 |
|------|----|----|
| Language support | English only | 30+ languages via mWhisper + multilingual LLM |
| Visual styles | Single cinematic style | 6 named style presets with dedicated prompt engines |
| Character consistency | Seed-based best-effort | Full Character Registry with IP-Adapter + LoRA locking |
| Storyboard editing | Scene strip only | Full drag-and-drop storyboard editor with custom prompt injection |
| Export | Single platform at a time | Parallel batch export to all platforms simultaneously |
| Scene regeneration | Full-scene only | Partial clip regeneration (trim window within a scene) |
| Processing feedback | Progress bar | Live storyboard preview — scenes appear as they generate |
| Infrastructure | 4 worker types | 7 worker types + dedicated style and character workers |

---

## Overview

**AI Reel Creator v2** expands the core V1 pipeline with four major capability layers: a **Style Preset Engine** that adapts every stage of generation to a named visual aesthetic; a **Character Consistency Module** that maintains coherent characters across all scenes; a **Multilingual Pipeline** supporting 30+ languages with culturally-aware scene planning; and a **Manual Storyboard Editor** that exposes the full shot plan to the user for direct editing before generation begins.

The V1 pipeline remains architecturally intact. V2 adds new modules and extends existing ones — it does not replace the V1 core.

---

## Updated System Architecture

### High-Level Pipeline (V2)

```
[Song File Input]
       │
       ▼
┌──────────────────────────────────┐
│  Audio Processing Module (APM)   │  ← Whisper v3 + mWhisper (multilingual)
│                                  │    + Demucs + librosa + essentia
└──────────────┬───────────────────┘
               │  { lyrics[], beats[], tempo, sections[], detected_language }
               ▼
┌──────────────────────────────────┐
│  Lyrics Intelligence Engine      │  ← Claude / GPT-4o (multilingual)
│  (LIE)                           │    + Cultural Context Adapter
└──────────────┬───────────────────┘
               │  { narrative, scene_plan[], emotional_arc, characters[] }
               ▼
┌──────────────────────────────────┐   ◄── NEW IN V2
│  Style Preset Engine (SPE)       │  ← Style adapter rewrites t2v prompts
│                                  │    per selected visual aesthetic
└──────────────┬───────────────────┘
               │  { styled_scene_plan[], style_tokens{}, lora_weights }
               ▼
┌──────────────────────────────────┐   ◄── UPGRADED IN V2
│  Character Consistency Module    │  ← IP-Adapter + LoRA fine-tuning
│  (CCM)                           │    + Character Registry
└──────────────┬───────────────────┘
               │  { character_references{}, consistency_seeds{} }
               ▼
┌──────────────────────────────────┐
│  Scene Generation Engine (SGE)   │  ← RunwayML Gen-3 + Stable Video
│                                  │    + SDXL + Real-ESRGAN
└──────────────┬───────────────────┘
               │  { scene_clips[], keyframes[] }
               ▼
┌──────────────────────────────────┐
│  Composition & Sync Engine (CSE) │  ← FFmpeg + beat-snap + lyric overlay
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐   ◄── UPGRADED IN V2
│  Batch Export & Delivery (BED)   │  ← Parallel platform encoding
│                                  │    + CDN + auto-thumbnail
└──────────────────────────────────┘
       │              │              │
       ▼              ▼              ▼
  Instagram        TikTok        YouTube
   Reels          Shorts          Shorts
```

### Manual Storyboard Editor — Optional Branch

After LIE + SPE complete, the user may intercept the pipeline:

```
LIE + SPE output
       │
       ├──► [AUTO MODE] ──► CCM ──► SGE ──► (continues)
       │
       └──► [EDITOR MODE] ──► Storyboard Editor UI
                                     │
                          User edits scenes, swaps shots,
                          rewrites prompts, adds custom media
                                     │
                                     ▼
                              Approved storyboard
                                     │
                                     ▼
                               CCM ──► SGE ──► (continues)
```

---

## Module Specifications

### 1. Audio Processing Module (APM) — V2 Upgrades

**New capabilities over V1:**
- Language auto-detection (ISO 639-1 code returned in output)
- Multilingual transcription via mWhisper and Whisper large-v3 multilingual checkpoint
- Romanization pass for CJK and Arabic scripts (feeds into LIE)
- Instrumental detection — flags songs with no lyrics and routes to energy-only mode

**Technology Stack:**

| Component | Tool | Purpose |
|-----------|------|---------|
| Vocal Separation | `Demucs` htdemucs_ft | Isolate clean vocal track |
| Transcription (English) | `Whisper large-v3` | Word-level timestamped lyrics |
| Transcription (multilingual) | `Whisper large-v3` multilingual checkpoint | 30+ language support |
| Romanization | `epitran` + `dragonmapper` | CJK/Arabic → Latin for LLM input |
| Beat Detection | `librosa` + `madmom` | BPM, beat grid, downbeats |
| Section Analysis | `essentia` MusicExtractor | Structural segmentation |
| Language Detection | `langdetect` + Whisper language token | Confirm detected language |

**Updated Output Schema (V2):**
```json
{
  "song_meta": {
    "title": "string",
    "bpm": 128,
    "duration_ms": 74000,
    "key": "Am",
    "time_signature": "4/4",
    "detected_language": "ko",
    "language_confidence": 0.97,
    "has_vocals": true
  },
  "lyrics": [
    {
      "word": "string",
      "word_romanized": "string",
      "start_ms": 4200,
      "end_ms": 4900,
      "line_index": 0,
      "section": "verse_1",
      "language": "ko"
    }
  ],
  "sections": [
    { "label": "intro", "start_ms": 0, "end_ms": 8000 },
    { "label": "verse_1", "start_ms": 8000, "end_ms": 24000 },
    { "label": "chorus", "start_ms": 24000, "end_ms": 40000 }
  ],
  "beat_grid": [0, 468, 937, 1406],
  "energy_curve": [0.2, 0.4, 0.8, 1.0],
  "instrumental_only": false
}
```

---

### 2. Lyrics Intelligence Engine (LIE) — V2 Upgrades

**New capabilities over V1:**
- **Multilingual narrative extraction** — prompts are now language-matched; Claude handles Korean, Spanish, French, Japanese, Hindi, Portuguese natively. All other languages receive English-translated lyrics as a parallel input.
- **Cultural Context Adapter (CCA)** — a lightweight prompt-layer that adjusts scene descriptions to be culturally appropriate to the song's language/region. A Korean ballad should not generate Western rural backdrops by default.
- **Character Registry seeding** — LIE now extracts named characters and their described appearance from lyrics, seeding the downstream Character Consistency Module.
- **Style-aware narrative mode** — LIE receives the selected style preset and adjusts narrative framing accordingly (anime style = more expressive, high-contrast emotional scenes; documentary style = grounded, realistic staging).

**Updated LLM Prompt Strategy (V2):**

**Step 1 — Multilingual Narrative Extraction:**
```
You are analyzing song lyrics to extract a visual narrative for a short-form video.

Lyrics language: {detected_language}
Lyrics (original): {lyrics_original}
Lyrics (english translation, if not english): {lyrics_translated}

Extract:
1. The core story (who, what, where, when, why)
2. All named or implied characters with physical/emotional descriptions
3. The emotional arc: list each section's dominant emotion
4. Key visual metaphors used by the lyricist
5. The cultural setting and time period implied
6. Any culturally specific visual references (festivals, landscapes, architecture)

IMPORTANT: Respect the cultural context of the song's origin language.
A J-Pop song implies Japanese visual references unless lyrics state otherwise.
A Punjabi folk song implies rural Punjab aesthetics unless stated otherwise.

Respond in structured JSON only.
```

**Step 2 — Style-Aware Scene Planning:**
```
Given:
- Story summary: {narrative}
- Song sections with timestamps: {sections}
- Selected visual style: {style_preset}  ← NEW in V2
- Target reel duration: {target_duration}s
- Character registry: {characters}  ← NEW in V2

Create a shot list. For each scene:
- scene_id, start_ms, end_ms
- lyric_lines (the specific lines this scene covers)
- description (what the viewer sees, framed for {style_preset} aesthetic)
- shot_type, camera_motion, mood
- color_palette (3 hex values matching {style_preset} color rules)
- t2v_prompt (optimized for RunwayML Gen-3, written in the voice of {style_preset})
- characters_present (list of character ids from registry appearing in this scene)
- is_key_scene (boolean — mark chorus/climax scenes true for higher generation budget)
```

**Character Registry Output (new in V2):**
```json
{
  "characters": [
    {
      "char_id": "c001",
      "name": "protagonist",
      "gender": "male",
      "age_range": "late 20s",
      "appearance": "worn leather jacket, dark hair, tired eyes",
      "emotional_role": "longing",
      "scenes_present": ["s001", "s003", "s005", "s008"]
    }
  ]
}
```

---

### 3. Style Preset Engine (SPE) — NEW IN V2

**Purpose:** Takes the raw scene plan from LIE and rewrites every visual prompt, color palette, camera directive, and transition style to conform to the selected aesthetic preset. This is not a simple filter — the SPE has dedicated prompt templates and model routing per style.

**Available Style Presets:**

| Preset | Visual Reference | Model Route | Color Rules | Transition Style |
|--------|-----------------|------------|-------------|-----------------|
| `cinematic` | Hollywood drama film | RunwayML Gen-3 | Desaturated, high contrast, teal-orange grade | Dissolve on beat |
| `anime` | Studio Ghibli / modern anime | NovelAI / Stable Diffusion anime checkpoint | Vivid, flat, high saturation | Hard cut + speed ramp |
| `illustration` | Editorial illustration / book cover | SDXL + illustrative LoRA | Muted pastels, geometric, hand-drawn feel | Page-turn wipe |
| `pixel_art` | 16-bit retro game | SDXL pixel LoRA + upscaler bypass | 32-color limited palette | Pixel dissolve |
| `documentary` | Handheld naturalistic film | RunwayML Gen-3 (realism prompt layer) | Natural, minimal grade, high dynamic range | Jump cut |
| `abstract` | Motion graphics / lyric video | Stable Diffusion abstract checkpoint | Gradient-heavy, symbolic, non-literal | Flow morph |

**SPE Processing Logic:**
```python
class StylePresetEngine:
    def __init__(self, preset: str):
        self.preset = STYLE_CONFIGS[preset]

    def rewrite_scene(self, scene: Scene) -> Scene:
        # 1. Inject style-specific prompt prefix and suffix
        scene.t2v_prompt = (
            self.preset.prompt_prefix +
            scene.t2v_prompt +
            self.preset.prompt_suffix
        )
        # 2. Override color palette with style palette
        scene.color_palette = self.preset.color_remap(scene.color_palette)
        # 3. Override camera motion if style has strong camera conventions
        if self.preset.force_camera_motion:
            scene.camera_motion = self.preset.camera_motion_for_mood(scene.mood)
        # 4. Override transition style
        scene.transition = self.preset.transition_style
        # 5. Set model routing
        scene.generation_model = self.preset.model_route
        return scene
```

**Style Config Schema:**
```json
{
  "preset_id": "anime",
  "display_name": "Anime",
  "description": "High-energy Japanese animation aesthetic",
  "prompt_prefix": "anime style, cel shading, vibrant colors, expressive faces, ",
  "prompt_suffix": ", Studio Ghibli quality, clean linework, detailed backgrounds",
  "model_route": "sdxl_anime_checkpoint",
  "lora_weights": ["anime_detail_v2", "ghibli_bg_v1"],
  "color_rules": {
    "saturation_boost": 1.4,
    "contrast_boost": 1.2,
    "grade_lut": "anime_vivid.cube"
  },
  "transition_style": "hard_cut",
  "force_camera_motion": false,
  "aspect_ratio": "9:16",
  "frame_rate": 24
}
```

---

### 4. Character Consistency Module (CCM) — NEW IN V2

**Purpose:** V1's approach (a shared seed + IP-Adapter reference) produced inconsistent characters between scenes, especially when camera angle or lighting changed. V2 introduces a dedicated CCM that generates a locked character reference set before any scene generation begins.

**Two-Phase Architecture:**

**Phase 1 — Character Reference Generation (pre-SGE):**
For each character in the registry, generate 4 reference images at different angles and lighting conditions. These become the locked visual identity for the character throughout all scene generation.

```python
def generate_character_reference(char: Character, style: StylePreset) -> CharacterRef:
    prompts = [
        f"{char.appearance}, front facing, {style.prompt_prefix}, neutral background",
        f"{char.appearance}, 3/4 view, {style.prompt_prefix}, neutral background",
        f"{char.appearance}, profile view, {style.prompt_prefix}, neutral background",
        f"{char.appearance}, close-up face, {style.prompt_prefix}, neutral background"
    ]
    reference_images = [sdxl_generate(p, seed=char.seed) for p in prompts]
    ip_adapter_embedding = ip_adapter.encode(reference_images)
    return CharacterRef(
        char_id=char.char_id,
        reference_images=reference_images,
        ip_adapter_embedding=ip_adapter_embedding,
        lora_path=None  # LoRA fine-tuning reserved for V2.1
    )
```

**Phase 2 — Consistency Injection (during SGE):**
Every scene that contains a character from the registry receives the character's IP-Adapter embedding injected into the generation call, at a weight tuned by scene type:

| Scene Type | IP-Adapter Weight | Reason |
|-----------|-----------------|--------|
| Close-up / portrait | 0.85 | High fidelity needed |
| Medium shot | 0.70 | Balance detail vs. scene context |
| Wide / establishing | 0.40 | Scene context dominates |
| Abstract / symbolic | 0.0 | Character not recognizable anyway |

**Character Registry Storage:**
Characters are stored in Redis with a 24-hour TTL matching the job lifecycle. Reference images are uploaded to S3 immediately and served from CDN for injection at generation time.

---

### 5. Scene Generation Engine (SGE) — V2 Upgrades

**New capabilities over V1:**
- Model routing per style preset (cinematic → RunwayML; anime → SDXL anime; pixel → SDXL pixel)
- Live scene streaming — each clip is pushed to the UI as it completes, not batched
- Partial regeneration — users can re-generate a 2-second window within a scene without re-generating the whole clip
- `is_key_scene` budget allocation — key scenes (chorus, climax) use 8-second generation at full resolution; non-key scenes use 4-second generation and Ken Burns if needed

**Updated Technology Stack (V2):**

| Component | Tool | V2 Change |
|-----------|------|-----------|
| Text-to-Video (cinematic, documentary) | `RunwayML Gen-3 Alpha` | Unchanged |
| Text-to-Video (anime, illustration) | `Stable Video Diffusion` + anime checkpoint | New in V2 |
| Image Generation | `Stable Diffusion XL 1.0` + ControlNet | Added LoRA weight loading |
| Style LoRAs | Custom trained per preset | New in V2 |
| Character Injection | `IP-Adapter Plus` (SDXL variant) | New in V2 |
| Upscaling | `Real-ESRGAN x4+` | Unchanged |
| Partial Regen | Custom inpainting pipeline | New in V2 |

**Updated Scene Duration Logic:**
```
scene_duration = end_ms - start_ms
is_key = scene.is_key_scene

if is_key and scene_duration >= 6000ms:   generate 8s clip, trim
if is_key and scene_duration < 6000ms:    generate 4s clip, trim
if not is_key and scene_duration <= 4000ms:  generate 4s clip, trim
if not is_key and scene_duration <= 8000ms:  generate 4s clip + Ken Burns extend
if not is_key and scene_duration > 8000ms:   two 4s clips with dissolve at midpoint
```

**Live Scene Streaming (V2):**
```
SGE worker completes scene s001
  → uploads clip to S3
  → emits WebSocket event: { type: "scene_ready", scene_id: "s001", clip_url: "...", thumbnail_url: "..." }
  → frontend inserts s001 into storyboard strip in real-time
  → continues generating s002, s003... in parallel (up to 3 concurrent)
```

---

### 6. Composition & Sync Engine (CSE) — V2 Upgrades

**New capabilities over V1:**
- Style-matched transition effects (anime preset uses hard cuts + speed ramps; illustration uses wipe transitions)
- Color grading LUT applied per style preset during composition, not post-processing
- Enhanced subtitle rendering — supports CJK characters, RTL scripts (Arabic, Hebrew), and custom font injection per style preset
- Energy-reactive transition timing — high-energy sections (chorus) get faster cuts; low-energy sections get longer dissolves

**Style-Matched Subtitle Fonts:**

| Preset | Subtitle Font | Size | Effect |
|--------|--------------|------|--------|
| `cinematic` | Helvetica Neue Light | 52px | Fade in/out |
| `anime` | Noto Sans JP Bold | 56px | Typewriter pop |
| `illustration` | Playfair Display | 48px | Handwrite draw |
| `pixel_art` | Press Start 2P | 32px | Scanline appear |
| `documentary` | Roboto Regular | 44px | Simple fade |
| `abstract` | Space Grotesk | 50px | Glitch reveal |

**Energy-Reactive Cut Algorithm:**
```python
def compute_cut_duration(scene: Scene, energy_at_beat: float) -> int:
    """Returns ideal scene duration in ms based on local energy level."""
    base_duration = scene.end_ms - scene.start_ms
    if energy_at_beat > 0.8:     # High energy (chorus, drop)
        return max(1500, base_duration * 0.7)   # Faster cuts
    elif energy_at_beat > 0.5:   # Mid energy (verse)
        return base_duration                     # As planned
    else:                         # Low energy (intro, outro)
        return min(8000, base_duration * 1.3)   # Longer, breathing cuts
```

---

### 7. Batch Export & Delivery (BED) — V2 Upgrade

**V1** exported to one platform at a time sequentially. **V2** runs all platform encodes in parallel using FFmpeg worker pool.

**Parallel Encoding Architecture:**
```
composed_reel.mp4  (master — 1080×1920, H.264, 60fps, AAC 256k)
        │
        ├──► [Worker A]  Instagram Reels encode  → instagram_reel.mp4
        ├──► [Worker B]  TikTok encode           → tiktok_short.mp4
        ├──► [Worker C]  YouTube Shorts encode   → youtube_short.mp4
        └──► [Worker D]  Universal 30fps encode  → universal_download.mp4
        
All four run in parallel. Total encode time: ~40s instead of ~160s.
```

**Updated Output Specifications (V2):**

| Platform | Resolution | FPS | Format | Max Duration | Audio | Thumbnail |
|----------|-----------|-----|--------|-------------|-------|-----------|
| Instagram Reels | 1080×1920 | 30 | MP4 H.264 | 90s | AAC 192k | Auto-selected from chorus |
| TikTok | 1080×1920 | 30 | MP4 H.264 | 60s | AAC 192k | Auto-selected from chorus |
| YouTube Shorts | 1080×1920 | 60 | MP4 H.264 | 60s | AAC 256k | Auto-selected from chorus |
| Universal DL | 1080×1920 | 30 | MP4 H.264 | 90s | AAC 192k | First frame of chorus |
| Master Archive | 1080×1920 | 60 | MP4 H.265 | 90s | AAC 256k | Stored for re-export |

**V2 adds a master H.265 archive** that is retained for 30 days (vs. 7 days for platform exports), enabling re-export with different settings without re-generating.

---

## Manual Storyboard Editor — NEW IN V2

The storyboard editor is an optional pre-generation intervention layer. After LIE + SPE produce the initial scene plan, users may enter the editor before any video generation begins.

### Editor Capabilities

| Feature | Description |
|---------|-------------|
| **Scene reorder** | Drag scenes to change their order |
| **Scene delete** | Remove scenes from the reel |
| **Scene duplicate** | Duplicate a scene (useful for repeating a visual motif) |
| **Prompt edit** | Rewrite any scene's t2v_prompt inline |
| **Shot type override** | Change close-up to wide, aerial, etc. |
| **Camera motion override** | Change motion (handheld, crane, static, etc.) |
| **Custom media inject** | Replace AI generation with user-uploaded image or video clip |
| **Duration adjust** | Drag scene endpoints on the timeline |
| **Character toggle** | Enable or disable character consistency per scene |
| **Split scene** | Split one scene at any timestamp into two independent scenes |
| **Lyric re-assign** | Change which lyric lines a scene illustrates |

### Editor State Schema
```json
{
  "job_id": "j_abc123",
  "editor_version": 2,
  "user_modified": true,
  "scenes": [
    {
      "scene_id": "s001",
      "start_ms": 8000,
      "end_ms": 12500,
      "lyric_lines": ["I walked the road we used to know"],
      "t2v_prompt": "User-edited prompt here...",
      "shot_type": "wide",
      "camera_motion": "slow push forward",
      "mood": "melancholic, warm",
      "color_palette": ["#D4A553", "#6B4226", "#E8D5A3"],
      "custom_media_url": null,
      "characters_present": ["c001"],
      "character_consistency_enabled": true,
      "is_key_scene": false,
      "user_locked": false,
      "generation_status": "pending"
    }
  ],
  "global_style": "cinematic",
  "approved_at": null
}
```

### Editor Auto-Save
The editor auto-saves the storyboard state to the server every 10 seconds. If the user closes the browser, they can resume editing from where they left off. Storyboard states are retained for 48 hours.

---

## Updated User Interface — V2 Screens

### 1. Upload Screen (updated)
- Drag-and-drop zone (unchanged)
- Language display — auto-detected language shown with flag + confidence
- **Style preset selector** — 6 visual styles with animated preview thumbnails (new)
- Duration selector: 30s / 45s / 60s / 90s (added 45s option)
- Section selector with waveform (unchanged)

### 2. Processing Screen (updated)
- Live storyboard preview — scenes appear as thumbnail cards as generation completes (new)
- Pipeline now shows 7 stages: APM → LIE → SPE → CCM → SGE → CSE → BED
- Character registry display — extracted characters shown with their reference images (new)
- Estimated time updates dynamically based on actual generation rate (new)
- **"Edit Storyboard" button** appears after SPE completes, before SGE begins (new)

### 3. Storyboard Editor Screen (new)
- Full-width scene strip with drag handles
- Each scene card: thumbnail (generated by SDXL keyframe), prompt text, duration badge, shot type tag
- Right panel: selected scene controls (all fields editable)
- Lyric sync view at bottom: shows lyrics timeline with scene ranges overlaid
- "Approve & Generate" CTA — locks the storyboard and begins SGE

### 4. Preview & Edit Screen (updated)
- Inline 9:16 video player (unchanged)
- Scene strip with per-scene regenerate (unchanged)
- **Partial regeneration** — drag a range handle within a scene to re-generate just that window (new)
- Color grade presets updated to 8 options (new: neon, vintage 8mm, golden era)
- **Style switcher** — change style preset post-generation and re-render (triggers full SGE re-run) (new)

### 5. Export Screen (updated)
- **Batch export toggle** — "Export to all platforms at once" (new)
- Per-platform status indicators (4 parallel encode progress bars) (new)
- Master archive download option (new)
- Share link with platform-specific deep links (new)

---

## Updated API Design

### New REST Endpoints (V2 additions)

```
# Style Presets
GET    /api/v2/styles                           # List all style presets with preview URLs
GET    /api/v2/styles/:id                       # Get full style config

# Storyboard Editor
GET    /api/v2/jobs/:id/storyboard              # Get current storyboard state
PUT    /api/v2/jobs/:id/storyboard              # Save edited storyboard (full replace)
PATCH  /api/v2/jobs/:id/storyboard/scenes/:sid  # Update a single scene
POST   /api/v2/jobs/:id/storyboard/scenes/:sid/split  # Split scene at timestamp
DELETE /api/v2/jobs/:id/storyboard/scenes/:sid  # Delete a scene
POST   /api/v2/jobs/:id/storyboard/approve      # Approve and begin SGE

# Character Registry
GET    /api/v2/jobs/:id/characters              # Get extracted characters + reference images
POST   /api/v2/jobs/:id/characters/:cid/regenerate  # Regenerate character reference images

# Partial Regeneration
POST   /api/v2/jobs/:id/scenes/:sid/regen-window  # Regen a time window within a scene
  body: { start_offset_ms: 1200, end_offset_ms: 3400 }

# Batch Export
POST   /api/v2/jobs/:id/export/batch            # Trigger parallel export to all platforms
GET    /api/v2/jobs/:id/export/batch/status     # Get per-platform encode status

# V1 endpoints remain at /api/v1/ for backwards compatibility
```

### Updated WebSocket Events (V2)

```
Events emitted by server:
  stage_update       { stage: "apm|lie|spe|ccm|sge|cse|bed", progress: 0-100, message: string }
  scene_ready        { scene_id: string, thumbnail_url: string, clip_url: string, is_key: bool }
  character_ready    { char_id: string, name: string, reference_images: string[] }
  storyboard_ready   { storyboard_url: string }  ← editor can now open
  reel_ready         { preview_url: string }
  export_platform    { platform: "instagram|tiktok|youtube|universal", status: "encoding|done|error", url?: string }
  error              { stage: string, code: string, message: string, recoverable: bool }
```

---

## Updated Processing Time Estimates (V2)

For a 60-second reel:

| Stage | V1 Time | V2 Time | Change |
|-------|---------|---------|--------|
| Audio Processing (APM) | 30–60s | 45–75s | +15s multilingual pass |
| Lyrics Intelligence (LIE) | 15–30s | 20–40s | +10s cultural context + character extraction |
| Style Preset Engine (SPE) | — | 5–10s | New |
| Character Consistency Module (CCM) | — | 60–90s | New — 4 reference images × N characters |
| Scene Generation (SGE) — 8–12 scenes | 4–8 min | 4–8 min | Unchanged per-scene; parallel helps |
| Composition & Sync (CSE) | 30–90s | 30–90s | Unchanged |
| Batch Export (BED) | 20–40s (×platforms) | 40–60s total | All platforms in parallel |
| **Total** | **~6–12 min** | **~8–14 min** | +2 min for CCM; batch export saves time |

*Note: CCM adds ~90s upfront but eliminates the need for multiple SGE retries due to character drift — net time is neutral to positive for reels with recurring characters.*

---

## Updated Infrastructure (V2)

```
User Browser
    │
    ▼
Next.js Frontend (Vercel Edge)
    │
    ▼
API Gateway (FastAPI / Python 3.12)
    │
    ├──► Job Queue (Redis 7 + BullMQ)
    │         │
    │         ├──► APM Worker          (GPU: g4dn.xlarge — Whisper + Demucs)
    │         ├──► LIE Worker          (CPU: c6i.large — LLM API calls)
    │         ├──► SPE Worker          (CPU: c6i.large — prompt rewriting, no GPU needed)
    │         ├──► CCM Worker          (GPU: g5.xlarge — IP-Adapter + SDXL ref gen)   ← NEW
    │         ├──► SGE Worker          (GPU: g5.2xlarge × 3 — parallel scene gen)
    │         ├──► CSE Worker          (CPU: c6i.2xlarge — FFmpeg composition)
    │         └──► BED Worker Pool     (CPU: c6i.xlarge × 4 — parallel encode)         ← NEW
    │
    ├──► Object Storage (S3)
    │     ├── /songs/         — input audio (24h TTL)
    │     ├── /references/    — character reference images (24h TTL)
    │     ├── /scenes/        — raw generated clips (24h TTL)
    │     ├── /reels/         — final platform exports (7-day TTL)
    │     └── /masters/       — H.265 archive masters (30-day TTL)       ← NEW
    │
    ├──► CDN (CloudFront) — streaming + download delivery
    │
    ├──► Redis                — job state, storyboard autosave, character registry (48h TTL)
    │
    └──► WebSocket Server (Socket.io) — live progress streaming to frontend
```

**Updated Compute Requirements:**

| Worker | Instance Type | GPU | Autoscale |
|--------|--------------|-----|-----------|
| APM | g4dn.xlarge | NVIDIA T4 | 1–3× |
| LIE | c6i.large | None | 1–5× |
| SPE | c6i.large | None | 1–5× |
| CCM | g5.xlarge | NVIDIA A10G | 1–3× |
| SGE | g5.2xlarge | NVIDIA A10G | 1–6× |
| CSE | c6i.2xlarge | None | 1–3× |
| BED | c6i.xlarge | None | 1–8× |

---

## Multilingual Support Matrix (V2)

| Language | Transcription | Scene Planning | Subtitle Render | Cultural Context |
|----------|--------------|---------------|----------------|-----------------|
| English | Whisper large-v3 | Native | Yes | Yes |
| Spanish | Whisper multilingual | Claude (native) | Yes | Yes |
| French | Whisper multilingual | Claude (native) | Yes | Yes |
| Korean | Whisper multilingual | Claude (native) | Yes (Noto Sans KR) | Yes |
| Japanese | Whisper multilingual | Claude (native) | Yes (Noto Sans JP) | Yes |
| Hindi | Whisper multilingual | Claude (translated) | Yes (Noto Sans Devanagari) | Yes |
| Portuguese | Whisper multilingual | Claude (native) | Yes | Yes |
| Arabic | Whisper multilingual | Claude (translated) | Yes (RTL, Noto Sans Arabic) | Yes |
| Mandarin | Whisper multilingual | Claude (translated) | Yes (Noto Sans SC) | Yes |
| German | Whisper multilingual | Claude (native) | Yes | Yes |
| 20+ others | Whisper multilingual | English translation path | Latin script | Partial |

*"Native" = LLM processes lyrics directly in that language without translation intermediary.*  
*"Translated" = APM produces an English translation; LIE operates on English with cultural notes.*

---

## Updated Component Library (V2)

### New Components

| Component | Description |
|-----------|-------------|
| `<StylePresetPicker>` | 6 preset tiles with animated preview loop, click to select |
| `<StoryboardEditor>` | Full drag-and-drop scene editing canvas |
| `<SceneEditorPanel>` | Right-side panel for editing a selected scene's properties |
| `<CharacterCard>` | Displays extracted character with 4-angle reference image grid |
| `<PartialRegenSlider>` | Drag handles within a scene clip for partial window regen |
| `<BatchExportStatus>` | 4-platform progress indicators running in parallel |
| `<LanguageDetectionBadge>` | Flag + language name + confidence score pill |
| `<LyricSyncTimeline>` | Dual-layer timeline: lyrics on top, scenes on bottom, linked |

### Updated Components (V1 → V2)

| Component | V1 | V2 Change |
|-----------|----|-----------| 
| `<ProcessingPipeline>` | 5 stages | 7 stages; live scene thumbnails stream in |
| `<SceneCard>` | Thumbnail + description | + generation status, + partial regen button |
| `<ExportPanel>` | Single platform | Batch toggle + 4 parallel status bars |
| `<VideoPreview>` | Basic player | + style switcher, + partial regen timeline overlay |

---

## Updated Design Token System (V2)

### Color Palette (extended for dark-first design)

```css
:root {
  /* Brand — unchanged */
  --color-brand-primary: #7C3AED;
  --color-brand-accent:  #F59E0B;

  /* Backgrounds — unchanged */
  --color-bg-base:     #0F0F14;
  --color-bg-surface:  #1A1A24;
  --color-bg-elevated: #242433;

  /* Text — unchanged */
  --color-text-primary:   #F5F5FA;
  --color-text-secondary: #9B99B4;
  --color-text-muted:     #5E5C7A;

  /* Style Preset Accent Colors — NEW in V2 */
  --style-cinematic:    #2D6A8F;
  --style-anime:        #E84393;
  --style-illustration: #7CB87C;
  --style-pixel:        #FF6B35;
  --style-documentary:  #8B7355;
  --style-abstract:     #9B59B6;

  /* Semantic — unchanged */
  --color-success:    #10B981;
  --color-warning:    #F59E0B;
  --color-error:      #EF4444;
  --color-processing: #6366F1;
}
```

---

## Error Handling (V2 additions)

| Error | Recovery Strategy |
|-------|------------------|
| Language detection ambiguous | Default to `und` (undetermined); use translation path; surface warning to user |
| Character reference generation fails | Disable CCM for that character; proceed without consistency; notify user |
| SPE returns invalid prompt (content filter) | Retry with safer prompt variant; if persists, revert to `cinematic` default |
| Storyboard state conflict (concurrent edit) | Last-write-wins with 10s debounce; show conflict notification in UI |
| RunwayML API timeout on key scene | Retry once at 8s; retry at 4s; fall back to SDXL keyframe + Ken Burns |
| Batch export partial failure | Failed platforms retry independently; others are not blocked |
| All V1 errors | V1 recovery strategies remain in place (unchanged) |

---

## Data Privacy & Rights (V2 additions)

- **Storyboard data:** User-edited storyboard states are stored for 48 hours and deleted automatically. No storyboard data is used for training.
- **Character reference images:** All AI-generated character reference images are deleted after 24 hours (with job cleanup). They are never used to train facial recognition or identity models.
- **Multilingual lyrics:** Translated lyrics are processed in memory only and not stored to disk. The original lyrics are retained only for the 24-hour job lifecycle.
- **Style LoRA weights:** Style LoRAs are trained on licensed datasets. No user-generated content is used to update style weights.

---

## Constraints & Limitations (V2)

| Limitation | Reason | V3 Plan |
|------------|--------|---------|
| LoRA fine-tuning per character not available | Requires 20+ min training time per character | V2.1 async LoRA fine-tune option |
| Style switching post-generation triggers full SGE re-run | Clips are style-baked at generation time | V3: style transfer post-processing |
| Pixel art max resolution 720p (upscaler artefacts at 1080p) | SDXL pixel LoRA trained at 512/768px | V3: dedicated pixel art pipeline |
| Arabic/Hebrew RTL subtitles require manual review | FFmpeg ASS RTL support is incomplete | V3: custom subtitle renderer |
| Storyboard editor not available on mobile | Complex drag-and-drop requires desktop | V2.1: simplified mobile editor |
| Max 1 active generation job per user | GPU quota management | V2.1: queue priority tiers |

---

## Roadmap

### V2.0 (Current — this document)
- Style Preset Engine (6 presets)
- Character Consistency Module
- Multilingual pipeline (30+ languages)
- Manual Storyboard Editor
- Batch parallel export
- Live scene streaming during generation
- Master archive with 30-day retention

### V2.1 (Next Quarter)
- Async LoRA fine-tuning for character consistency
- Simplified mobile storyboard editor
- Queue priority tiers (free / pro / enterprise)
- Arabic/Hebrew RTL subtitle renderer
- Style transfer post-processing (no full re-gen needed for style switch)

### V3 (Future)
- Real-time collaboration — share WIP reel with teammates; multiplayer storyboard editing
- Brand kit integration — logo watermark, brand color palette auto-injected per preset
- Music library integration — licensed tracks for commercial use, royalty-cleared
- Auto-publishing to Instagram/TikTok/YouTube via OAuth
- Video-to-reel mode — upload an existing video and re-style it to match a song
- Live performance mode — generate real-time visuals from a live audio stream

---

*End of DESIGN.md v2.0*
