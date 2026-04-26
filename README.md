# Motif — AI Music Video Generator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Motif is an AI-powered short-form music video generator. Given a song, it plans a narrative from the lyrics, generates visuals, syncs cuts to the beat, and outputs a 9:16 reel ready for Instagram Reels, TikTok, and YouTube Shorts.

> **Project status:** Pre-development. This repository currently contains design, requirements, and task planning documents for Motif’s V1/V2 roadmap.

## What the project does
- Transcribes lyrics and detects beats/sections from an audio file.
- Uses an LLM to generate a story-driven scene plan from lyrics.
- Generates visuals per scene with AI image/video models.
- Composes a beat-synced reel with lyric subtitles.
- Exports platform-ready 9:16 MP4 files.

## Why it’s useful
- **Fast content creation**: turn a song into a shareable reel in minutes.
- **Story-driven output**: visuals map to the lyrical narrative, not just mood.
- **Beat-accurate editing**: transitions align to the song’s tempo.
- **Flexible styles**: design supports cinematic, anime, illustration, pixel art, documentary, and abstract presets (V2).

## Getting started
Since the codebase is not yet implemented, the quickest way to get productive today is to review the project documents and plan your implementation:

1. Clone the repository:
   ```bash
   git clone https://github.com/anan5093/Motif-AI-Music-Video-Generator.git
   cd Motif-AI-Music-Video-Generator
   ```
2. Read the core docs:
   - [Requirements](Requirements.md) — functional and non-functional requirements.
   - [Design](Design.md) — V2 architecture and system design.
   - [Task plan](Task.md) — phased task breakdown and roadmap.

### Planned local setup (once implemented)
The V1 plan is a FastAPI backend + Next.js frontend with FFmpeg and optional GPU acceleration. Expected local prerequisites:
- **Python** 3.11+
- **Node.js** 20+
- **FFmpeg** 5.0+
- **GPU (optional)** for SDXL/Whisper large-v3

### Planned usage example (API preview)
These endpoints are specified in [Requirements](Requirements.md) and are not yet implemented:

```bash
# Create a job
curl -X POST http://localhost:8000/api/v1/jobs \
  -F "file=@song.mp3" \
  -F "duration_s=60" \
  -F "style=cinematic"

# Check status
curl http://localhost:8000/api/v1/jobs/j_abc123

# Download result
curl -L http://localhost:8000/api/v1/jobs/j_abc123/download -o motif_j_abc123.mp4
```

## Project structure (planned)
```
motif/
├── backend/        # FastAPI pipeline (APM → LIE → SGE → CSE)
├── frontend/       # Next.js UI (upload → processing → result)
└── docs/           # Architecture, testing, and deployment notes
```

## Documentation
- **Requirements:** [Requirements.md](Requirements.md)
- **System design (V2):** [Design.md](Design.md)
- **Roadmap & tasks:** [Task.md](Task.md)

## Getting help
- Open a GitHub issue for questions or suggestions.
- Review the requirements/design docs before proposing changes.

## Contributing
Contributions are welcome as the implementation begins. For now:
- Start with [Task.md](Task.md) to pick a task.
- Use clear commit messages and open a PR with context and screenshots (if UI-related).

## Maintainer
- **Anand Raj** (project author)

## License
MIT — see [LICENSE](LICENSE).
