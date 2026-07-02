# ARCHITECTURE.md — Voiden Cinematic Engine

> **Document Version**: 2.0  
> **Engine Version**: V31.3  
> **Last Updated**: 2026-07-02

---

## 1. System Overview

Voiden Cinematic Engine is a **scheduler-based, deterministic video editing pipeline** that transforms raw footage and background music into algorithmically edited cinematic shorts.

### Design Philosophy

- **Content-Agnostic Core**: Audio analysis, video metrics, and render engine work with any footage type
- **Template-Based Specialization**: Content-specific behavior (gaming vs travel vs sports) injected via JSON templates
- **Determinism**: Same input → same output (with optional seed mode)
- **Minimal Overlap**: Each pipeline stage has a single responsibility
- **Single Source of Truth**: `output_events` drives all downstream timing
- **Scheduler-Based**: No heuristic layering; central timing authority

---

## 2. Content Type Abstraction

The engine uses a **template-based abstraction layer** to support multiple content types without core code changes:

```
┌─────────────────┐
│  Content Type   │
│   Template      │
│  (JSON config)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Asset Scanner  │──▶ Filters by type (motion threshold, duration range)
│  (type-aware)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Identity Layer │──▶ Mood clusters adapt per content type
│  (adaptive)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Render Engine  │──▶ Effect presets per type (zoom/shake/transition)
│  (preset-based) │
└─────────────────┘
```

### Current Template: Gaming
- **Asset filter**: `.mp4` with motion intensity > 0.3, chaos < 0.92
- **Identity clusters**: tragic, power, ascension, revenge, lonely
- **Effects**: Impact flash, shake, slow-mo, progressive zoom
- **Subtitle style**: aggressive / melancholic / ascension
- **Upload defaults**: category "Gaming", tags ["gaming", "shorts"]

### Future Templates
| Template | Key Differences |
|----------|----------------|
| **Travel** | Smooth transitions, no shake, scenic hold, calm cadence |
| **Sports** | Fast cuts, high motion tolerance, crowd audio mixing, energetic cadence |
| **Music** | Lyric sync, beat grid, visualizer overlay, rhythmic cadence |
| **Fashion** | Elegant cuts, product focus, slow zoom, luxurious cadence |

---

## 3. Pipeline Stages

### Stage 1: Asset Ingestion

**Input**: Raw video (MP4/MOV) + raw music (MP3/M4A/WAV/FLAC/OGG)

**Process**:
1. Scan `videos/` and `music/` directories (template defines subfolders)
2. Filter by template criteria: min duration, motion threshold, format
3. Select assets using cycle tracking (no cooldown, round-robin with reset)
4. Extract video track → `video_extract.mp4`
5. Extract audio track → `audio_extract.wav`

**Output**: Extracted video + audio files

---

### Stage 2: Audio Analysis

**Input**: Music file + extracted footage audio

**Process**:
1. **Decode** to PCM16 WAV at 22.05kHz (cached by hash)
2. **Build envelopes**: RMS, peak, onset at 10ms frame / 2ms hop
3. **Detect music peaks**: Combined onset×0.30 + peak×0.20 + RMS×0.50, filtered by min-gap
4. **Detect footage impacts**: Combined onset×0.75 + peak×0.20 + RMS×0.25, with body/sustain analysis
5. **Section cycling**: Filter peaks/impacts by bucketed history (template-defined bucket size)

**Output**:
- `energy_time`: Primary music peak
- `impact_time_source`: Primary footage impact
- `smart_cut`: Optimal cut point within template-defined window
- `events`: Post-impact transient events (template-defined max count)
- `candidate_pool`: Ranked video-audio pairs (up to 16)

---

### Stage 3: Candidate Ranking

**Input**: Footage impacts + audio peaks + video duration

**Process**:
1. **Build pairs**: Cartesian product of impacts × peaks
2. **Sample video**: Deterministic per-pair segment for motion analysis
3. **Compute metrics**:
   - Motion intensity (optical flow median)
   - Motion stability (directional coherence)
   - Clarity (Laplacian variance)
   - Chaos score (incoherence + temporal instability)
   - Motion variation & peak contrast
4. **Pool normalization**: Relative scores within candidate pool
5. **Diversity scoring**: Distance from all other candidates
6. **Final score**: 9 weighted components with chaos/hook penalties
7. **Visual confidence**: Boost/penalty based on visual relevance (template-adjusted threshold)

**Output**: Ranked candidate pool with debug breakdowns

---

### Stage 4: Segment Timeline

**Input**: Candidate pool + effect plan (hook_time, energy_time, impact_time, smart_cut)

**Process**:
1. **Estimate content duration**: `smart_cut - (energy_time - hook_time)`
2. **Determine segment count**: `content_duration / 2.35` (min 3, template-adjustable)
3. **Assign roles**: HOOK → BUILD → MOTION → IMPACT → RELEASE (repeating, template can override)
4. **Select candidates**: Score = base_score + role_fit - penalty + novelty
5. **Fatigue tracking**: Accumulates when same role/action/cut-point repeats (template-adjusted sensitivity)
6. **Beat-aligned cuts**: Snap segment ends to event anchors (±0.35s tolerance, template-adjustable)
7. **Mandatory release**: Final segment forced to RELEASE if ends on IMPACT/MOTION (template can disable)

**Output**: Segment timeline with transitions, roles, and candidate metadata

---

### Stage 5: Identity Layer

**Input**: Effect plan + audio/video metadata + subtitles + **template config**

**Process**:
1. **Tokenize** filenames for lexical nudges (template defines cluster set)
2. **Compute weights**: Based on event density, tail ratio, strength distribution, spacing
3. **Normalize** to probability distribution
4. **Derive behaviors** (template overrides available):
   - Subtitle profile (melancholic/aggressive/ascension/calm/energetic/luxurious)
   - Visual intensity preference
   - Silence spacing
   - Motion recovery speed
5. **Build outro signature**: Cadence name, pulse pattern, text alpha/position/scale
6. **Select opening text**: Deterministic from template-defined `OPENING_TEXTS` array

**Output**: Identity profile dict driving all stylistic decisions

---

### Stage 6: Render Engine

**Input**: Video extract + effect plan + segment timeline + identity profile + subtitles + **template presets**

**Process**:
1. **Time remapping**: Binary search for slowmo start to hit exact hook_time (template can disable)
2. **Frame-by-frame processing** (template toggles effects):
   - Progressive zoom (1.0 → ZOOM_MAX → 1.0) [template: on/off]
   - Impact shake (random shift, strength-based amplitude) [template: on/off]
   - Flash overlay (white blend, strength-based) [template: on/off]
   - Reseed text injection (mid-spike, clutch, escalation) [template: on/off]
   - Opening text (template-defined time window)
   - Dark fade (template-defined start percentage)
   - Ending text (on black, pulse-cadenced)
   - Subtitles (post-fade, identity-styled)
3. **Audio retiming**: Warp footage audio to match remapped time
4. **Mux**: Combine video + retimed footage audio with volume expression
5. **Finalize**: Mix footage audio + music with ducking, fade in/out, limiter

**Output**: `video_content.mp4` (1080×1920, H.264/AAC)

---

### Stage 7: Upload

**Input**: Final video + metadata JSON + **template upload defaults**

**Process**:
1. **Hash check**: SHA-256 deduplication + 24h cooldown
2. **Platform selection**: YouTube (default) + Facebook (optional)
3. **YouTube**: OAuth2 flow, resumable upload, privacy status (template default)
4. **Facebook**: Graph API feed upload or 3-step story upload
5. **Retry**: 2 attempts per platform with 5s delay
6. **Mark uploaded**: Update JSON DB with platform + remote_id

**Output**: Upload confirmation with video IDs

---

## 4. Data Flow Diagram

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Raw Video  │───▶│   Extract   │───▶│   Analyze   │───▶│   Build     │
│  Raw Music  │    │   Tracks    │    │   Audio     │    │  Candidate  │
└─────────────┘    └─────────────┘    └─────────────┘    │    Pool     │
                                                          └──────┬──────┘
                                                                 │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐           │
│   Upload    │◀───│   FFmpeg    │◀───│   Render    │◀──────────┘
│  (YouTube)  │    │   Finalize  │    │   Frames    │
└─────────────┘    └─────────────┘    └─────────────┘
       ▲                                    ▲
       │                                    │
┌──────┴──────┐                      ┌──────┴──────┐
│  Identity   │─────────────────────▶│   Reseed    │
│   Layer     │                      │  Scheduler  │
└─────────────┘                      └─────────────┘
       ▲
       │
┌──────┴──────┐
│   Template  │
│   Config    │
└─────────────┘
```

---

## 5. Key Algorithms

### 5.1 Time Remapping

Binary search for slowmo start position so that impact_source_time maps exactly to hook_time in output space.

```
find_slowmo_start_binary(target_hook_time, impact_source_time, duration, slowmo_min_speed)
  → slowmo_start_rel
```

Speed curve: `ease = 1 - (1 - progress)³`, applied between slowmo_start and impact_time.

### 5.2 Candidate Scoring

```
final_score = (
  audio_contrib(0.18) +
  visual_contrib(0.32) +
  motion_contrib(0.07) +
  variation_contrib(0.07) +
  stability_contrib(0.14) +
  clarity_contrib(0.08) +
  chaos_contrib(0.08) +
  hook_contrib(0.04) +
  diversity_contrib(0.02)
) × bounded_penalty × 100
```

### 5.3 Fatigue System

```
fatigue_increment = (
  same_action × 0.28 +
  same_role × 0.24 +
  close_cut × 0.18 +
  low_variation × 0.16 +
  similar_clarity × 0.08 +
  low_novelty × 0.18
)

fatigue_score = clamp01(fatigue_score × 0.58 + fatigue_increment)

if fatigue_score ≥ 0.72:
  force_role_refresh()
  fatigue_score ×= 0.45
```

---

## 6. Configuration Hierarchy

```
config.py (constants & defaults)
    │
    ├── template: templates/<type>_v1.json
    │   └── content_type, asset_filters, effect presets, upload defaults
    │
    ├── runtime: platform_upload_config.json
    │   └── targets, privacy_status, tokens
    │
    ├── per-asset: asset_tracker.json
    │   └── usage counts, last_used, cycle state
    │
    ├── per-session: reseed_config.json
    │   └── texts, intervals, gaps, weights
    │
    └── per-job: metadata JSON
        └── title, description, tags
```

---

## 7. Error Handling Strategy

| Layer | Strategy |
|-------|----------|
| Pre-flight | Disk space, FFmpeg availability, lock file, template validation |
| Asset selection | 10 tries with auto-reset cycle, template filter fallback |
| Render | 3 retries with cleanup between attempts |
| Upload | 2 retries, validate remote_id before marking |
| Fatal | Log + heartbeat + sys.exit(1) |

---

## 8. Performance Characteristics

| Metric | Value |
|--------|-------|
| Typical render time | 2-5 minutes per video (template-dependent) |
| Memory footprint | ~2GB peak (OpenCV + FFmpeg) |
| CPU priority | Below normal (Windows) / nice 10 (Unix) |
| FFmpeg threads | 2 |
| OpenCV threads | 1 |
| Throttle | 2ms sleep per frame |

---

## 9. Extensibility Points

### 9.1 Adding a New Content Type

1. Create `templates/<type>_v1.json` with:
   - Asset filters (motion threshold, duration range, format)
   - Identity cluster set
   - Effect presets (which effects on/off)
   - Upload defaults (category, tags)
2. Add template loader in `main.py`
3. Update README roadmap table

### 9.2 Adding a New Platform

1. Add to `SUPPORTED_TARGETS` in `uploader.py`
2. Implement `upload_to_<platform>()` function
3. Add config section in `platform_upload_config.json`
4. Update `upload_to_platform()` dispatcher

### 9.3 Adding a New Effect

1. Add parameter to `config.py`
2. Add toggle in template JSON
3. Implement in `video_processor.py` frame loop
4. Add to `effect_plan` contract validation
5. Update `identity_layer` if style-dependent
