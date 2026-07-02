# ARCHITECTURE.md — Voiden Cinematic Engine

> **Document Version**: 1.0  
> **Engine Version**: V31.3  
> **Last Updated**: 2026-07-02

---

## 1. System Overview

Voiden Cinematic Engine is a **scheduler-based, deterministic video editing pipeline** that transforms raw gameplay footage and background music into algorithmically edited cinematic shorts.

### Design Philosophy

- **Determinism**: Same input → same output (with optional seed mode)
- **Minimal Overlap**: Each pipeline stage has a single responsibility
- **Single Source of Truth**: `output_events` drives all downstream timing
- **Scheduler-Based**: No heuristic layering; central timing authority

---

## 2. Pipeline Stages

### Stage 1: Asset Ingestion

**Input**: Raw video (MP4/MOV) + raw music (MP3/M4A/WAV/FLAC/OGG)

**Process**:
1. Scan `videos/cinematic/` and `music/` directories
2. Filter by minimum duration (default: 8s)
3. Select assets using cycle tracking (no cooldown, round-robin with reset)
4. Extract video track → `video_extract.mp4`
5. Extract audio track → `audio_extract.wav`

**Output**: Extracted video + audio files

---

### Stage 2: Audio Analysis

**Input**: Music file + extracted game audio

**Process**:
1. **Decode** to PCM16 WAV at 22.05kHz (cached by hash)
2. **Build envelopes**: RMS, peak, onset at 10ms frame / 2ms hop
3. **Detect music peaks**: Combined onset×0.30 + peak×0.20 + RMS×0.50, filtered by min-gap
4. **Detect game impacts**: Combined onset×0.75 + peak×0.20 + RMS×0.25, with body/sustain analysis
5. **Section cycling**: Filter peaks/impacts by bucketed history (2s for video, 12s for audio)

**Output**:
- `energy_time`: Primary music peak
- `impact_time_source`: Primary game impact
- `smart_cut`: Optimal cut point within 25-30s window
- `events`: Post-impact transient events (up to 12)
- `candidate_pool`: Ranked video-audio pairs (up to 16)

---

### Stage 3: Candidate Ranking

**Input**: Video impacts + audio peaks + video duration

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
7. **Visual confidence**: Boost/penalty based on visual relevance

**Output**: Ranked candidate pool with debug breakdowns

---

### Stage 4: Segment Timeline

**Input**: Candidate pool + effect plan (hook_time, energy_time, impact_time, smart_cut)

**Process**:
1. **Estimate content duration**: `smart_cut - (energy_time - hook_time)`
2. **Determine segment count**: `content_duration / 2.35` (min 3)
3. **Assign roles**: HOOK → BUILD → MOTION → IMPACT → RELEASE (repeating)
4. **Select candidates**: Score = base_score + role_fit - penalty + novelty
5. **Fatigue tracking**: Accumulates when same role/action/cut-point repeats
6. **Beat-aligned cuts**: Snap segment ends to event anchors (±0.35s tolerance)
7. **Mandatory release**: Final segment forced to RELEASE if ends on IMPACT/MOTION

**Output**: Segment timeline with transitions, roles, and candidate metadata

---

### Stage 5: Identity Layer

**Input**: Effect plan + audio/video metadata + subtitles

**Process**:
1. **Tokenize** filenames for lexical nudges (5 clusters: tragic, power, ascension, revenge, lonely)
2. **Compute weights**: Based on event density, tail ratio, strength distribution, spacing
3. **Normalize** to probability distribution
4. **Derive behaviors**:
   - Subtitle profile (melancholic/aggressive/ascension)
   - Visual intensity preference
   - Silence spacing
   - Motion recovery speed
5. **Build outro signature**: Cadence name, pulse pattern, text alpha/position/scale
6. **Select opening text**: Deterministic from `OPENING_TEXTS` array

**Output**: Identity profile dict driving all stylistic decisions

---

### Stage 6: Render Engine

**Input**: Video extract + effect plan + segment timeline + identity profile + subtitles

**Process**:
1. **Time remapping**: Binary search for slowmo start to hit exact hook_time
2. **Frame-by-frame processing**:
   - Progressive zoom (1.0 → ZOOM_MAX → 1.0)
   - Impact shake (random shift, strength-based amplitude)
   - Flash overlay (white blend, strength-based)
   - Reseed text injection (mid-spike, clutch, escalation)
   - Opening text (0.5-3.5s)
   - Dark fade (75% duration → blackout)
   - Ending text (on black, pulse-cadenced)
   - Subtitles (post-75%, identity-styled)
3. **Audio retiming**: Warp game audio to match remapped time
4. **Mux**: Combine video + retimed game audio with volume expression
5. **Finalize**: Mix game audio + music with ducking, fade in/out, limiter

**Output**: `video_content.mp4` (1080×1920, H.264/AAC)

---

### Stage 7: Upload

**Input**: Final video + metadata JSON

**Process**:
1. **Hash check**: SHA-256 deduplication + 24h cooldown
2. **Platform selection**: YouTube (default) + Facebook (optional)
3. **YouTube**: OAuth2 flow, resumable upload, privacy status
4. **Facebook**: Graph API feed upload or 3-step story upload
5. **Retry**: 2 attempts per platform with 5s delay
6. **Mark uploaded**: Update JSON DB with platform + remote_id

**Output**: Upload confirmation with video IDs

---

## 3. Data Flow Diagram

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
```

---

## 4. Key Algorithms

### 4.1 Time Remapping

Binary search for slowmo start position so that impact_source_time maps exactly to hook_time in output space.

```
find_slowmo_start_binary(target_hook_time, impact_source_time, duration, slowmo_min_speed)
  → slowmo_start_rel
```

Speed curve: `ease = 1 - (1 - progress)³`, applied between slowmo_start and impact_time.

### 4.2 Candidate Scoring

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

### 4.3 Fatigue System

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

## 5. Configuration Hierarchy

```
config.py (constants & defaults)
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

## 6. Error Handling Strategy

| Layer | Strategy |
|-------|----------|
| Pre-flight | Disk space, FFmpeg availability, lock file |
| Asset selection | 10 tries with auto-reset cycle |
| Render | 3 retries with cleanup between attempts |
| Upload | 2 retries, validate remote_id before marking |
| Fatal | Log + heartbeat + sys.exit(1) |

---

## 7. Performance Characteristics

| Metric | Value |
|--------|-------|
| Typical render time | 2-5 minutes per video |
| Memory footprint | ~2GB peak (OpenCV + FFmpeg) |
| CPU priority | Below normal (Windows) / nice 10 (Unix) |
| FFmpeg threads | 2 |
| OpenCV threads | 1 |
| Throttle | 2ms sleep per frame |
