# Voiden Cinematic Engine

> **Universal Cinematic Video Engine**  
> Transform any raw footage — gameplay, travel, sports, music — into algorithmically edited cinematic shorts.  
> Fully automated: from scene analysis to platform upload.

---

## 🎬 What It Does

Voiden Cinematic Engine is a **content-type agnostic** video editing pipeline that:

1. **Analyzes** raw footage + background music
2. **Detects** audio peaks, transient impacts, and motion patterns
3. **Ranks** thousands of candidate cut points using multi-factor scoring
4. **Builds** a segment timeline with role-based sequencing (HOOK → BUILD → MOTION → IMPACT → RELEASE)
5. **Renders** slow-motion, zoom, flash, shake, and subtitle overlays synced to the beat
6. **Uploads** finished videos to YouTube and Facebook automatically

### Content Types Supported

| Content Type | Status | Description |
|-------------|--------|-------------|
| **Gaming** | ✅ Live | Slow-mo synced to impact moments, shake, flash |
| **Travel Vlog** | 🔄 In Progress | Smooth transitions, scenic holds, mood pacing |
| **Sports** | 📋 Planned | Peak action detection, dynamic zoom, crowd audio |
| **Music Video** | 📋 Planned | Lyric-synced visuals, beat grid, visualizer overlay |
| **Fashion/Lifestyle** | 📋 Planned | Elegant cuts, product focus, trend pacing |

> **Note**: The engine architecture is fully content-agnostic. Gaming is the first vertical to validate the pipeline. Universal templates are in active development.

---

## 🏗️ Architecture Overview

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Raw Assets     │────▶│  Audio/Video     │────▶│  Impact & Peak  │
│  (Any Footage)  │     │  Extraction      │     │  Detection      │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  YouTube / FB   │◀────│  FFmpeg Render   │◀────│  Candidate      │
│  Upload         │     │  + Subtitle Mix  │     │  Pool Builder   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │                        │
                               ▼                        ▼
                        ┌──────────────┐        ┌─────────────────┐
                        │  Slow-Mo +   │        │  Segment        │
                        │  Time Remap  │        │  Timeline       │
                        └──────────────┘        └─────────────────┘
                               │
                               ▼
                        ┌──────────────┐
                        │  Identity    │
                        │  Layer       │
                        │  (Mood/Style)│
                        └──────────────┘
```

### Core Pipeline Stages

| Stage | Responsibility |
|-------|---------------|
| **Audio Analysis** | Onset detection, RMS envelope, peak extraction with section cycling |
| **Video Analysis** | Optical flow, motion intensity, clarity, chaos scoring via OpenCV |
| **Candidate Ranking** | Multi-factor scoring: audio alignment, visual strength, hook timing, novelty, diversity |
| **Segment Timeline** | Role-based sequencing with fatigue system and beat-aligned cuts |
| **Identity Layer** | Emotional profiling (tragic/power/ascension/revenge/lonely) driving subtitle style & outro cadence |
| **Render Engine** | Time-remapped slowmo, progressive zoom, impact flash, shake, subtitle burn |
| **Upload** | YouTube Data API v3 + Facebook Graph API with cooldown & deduplication |

---

## 📊 Key Metrics

- **Detection**: ~16 impact candidates per audio track, filtered to top-ranked pairs
- **Scoring**: 9 component scores (audio, visual, motion, stability, clarity, chaos, hook, diversity, novelty)
- **Timeline**: 3-7 segments per video with automatic role progression
- **Render**: FFmpeg-based with NVENC/CPU fallback, AAC 192kbps
- **Upload**: Automatic hash-based deduplication, 24h cooldown, retry logic

---

## 🛠️ Tech Stack

| Layer | Technology | Content-Agnostic? |
|-------|-----------|-------------------|
| Language | Python 3.11+ | ✅ |
| Video Processing | OpenCV + FFmpeg | ✅ |
| Audio Analysis | NumPy + FFT-based envelope detection | ✅ |
| Text Rendering | Pillow (multi-script: Latin, CJK, Hangul, Cyrillic) | ✅ |
| Upload APIs | YouTube Data API v3, Facebook Graph API | ✅ |
| Storage | JSON-based tracker (scene history, asset cycles, upload DB) | ✅ |

---

## 🎥 Demo

> **[▶ Watch Demo](https://drive.google.com/file/d/13Dy9U5q8wHzKcFqGW17jvfr5aygP3zeW/view?usp=drive_link)**

The demo shows:
1. Terminal output: peak detection scores, candidate ranking, timeline build
2. Render progress: slowmo parameters, zoom curve, subtitle timing
3. Final output: 1080×1920 cinematic short with synced effects

---

## 📁 Repo Structure

```
voiden-cinematic-engine/
├── README.md              ← This file
├── .gitignore             ← Excludes all source code & credentials
├── docs/
│   ├── ARCHITECTURE.md    ← Detailed pipeline documentation
│   └── SYSTEM_DESIGN.md   ← Design decisions & state machines
├── src/
│   └── .gitkeep           ← Source code lives in private repo
├── demo/
│   └── .gitkeep           ← Demo video or link
└── assets/
    └── (architecture diagrams)
```

> **Note**: Source code is kept private. This public repo contains architecture documentation only.

---

## 🚧 Current Status & Roadmap

| Milestone | Status | Target |
|-----------|--------|--------|
| Gaming template (proof-of-concept) | ✅ Complete | Q2 2026 |
| Travel vlog template | 🔄 In Progress | Q3 2026 |
| Sports template | 📋 Planned | Q4 2026 |
| Music video template | 📋 Planned | Q4 2026 |
| Fashion/lifestyle template | 📋 Planned | Q1 2027 |
| Web dashboard for template selection | 📋 Planned | Q1 2027 |

---

## 🔒 Implementation

Architecture and system design are documented here.  
Implementation details can be discussed via **screen share** — feel free to reach out.

For collaboration inquiries, please contact via GitHub issues or email.

## 📝 License

Private / All Rights Reserved.  
Architecture documentation © 2026 Faqihurrahman.
