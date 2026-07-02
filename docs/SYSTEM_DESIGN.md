# SYSTEM_DESIGN.md — Voiden Cinematic Engine

> **Document Version**: 2.0  
> **Engine Version**: V31.3  
> **Last Updated**: 2026-07-02

---

## 1. Design Principles

### 1.1 Content-Agnostic Core

The engine core (audio analysis, video metrics, render engine, upload) is **fully content-agnostic**. Content-specific behavior is injected via **JSON templates**, not hardcoded logic.

```
Core Engine (unchanged for all types)
    │
    ├── Template: Gaming
    │   └── High motion, impact flash, shake, aggressive cadence
    │
    ├── Template: Travel
    │   └── Smooth transitions, scenic hold, calm cadence
    │
    └── Template: Sports
        └── Fast cuts, crowd audio, energetic cadence
```

### 1.2 Scheduler-Based Architecture

The engine rejects heuristic layering in favor of a **central timing authority**. All timing decisions flow from a single source:

- `output_events` → drives render frame decisions
- `segment_timeline` → drives candidate selection
- `effect_plan` → drives audio trimming and time remapping

No parallel logic paths. No competing timing systems.

### 1.3 Determinism

With `SEED_MODE = True`, the entire pipeline produces identical output for identical input:
- Random choices use fixed seed (42)
- Hash-based stable units for identity layer
- Deterministic sampling plans per candidate pair

### 1.4 Minimal Overlap

Each module has exactly one responsibility:

| Module | Responsibility | Never Does |
|--------|---------------|-----------|
| `audio_analyzer` | Peak/impact detection | Render decisions |
| `segment_selector` | Candidate scoring | Timeline building |
| `momentum_engine` | Timeline sequencing | Video analysis |
| `identity_layer` | Style profiling | Timing calculations |
| `video_processor` | Frame rendering | Audio analysis |
| `uploader` | Platform upload | Video processing |
| `template_loader` | Config injection | Core logic |

---

## 2. Template System Design

### 2.1 Template Configuration (JSON)

```json
{
  "template_id": "gaming_v1",
  "content_type": "gaming",
  "version": "1.0",

  "asset_filters": {
    "min_duration": 8.0,
    "max_duration": 300.0,
    "min_motion_intensity": 0.3,
    "max_chaos_score": 0.92,
    "preferred_resolution": [1080, 1920],
    "supported_formats": [".mp4", ".mov"]
  },

  "timing": {
    "hook_time": 3.0,
    "smart_cut_min_offset": 25.0,
    "smart_cut_max_offset": 30.0,
    "event_window_after_impact": 27.0,
    "min_tail": 8.0
  },

  "identity_clusters": ["tragic", "power", "ascension", "revenge", "lonely"],

  "effect_presets": {
    "impact_flash": true,
    "shake": true,
    "slowmo": true,
    "progressive_zoom": true,
    "dark_fade": true,
    "subtitle_burn": true
  },

  "subtitle_profiles": ["melancholic", "aggressive", "ascension"],

  "upload_defaults": {
    "category_id": "20",
    "privacy_status": "public",
    "tags": ["gaming", "shorts", "cinematic"]
  }
}
```

### 2.2 Template Loader

```python
# Pseudocode
class TemplateLoader:
    def load(self, template_id: str) -> Template:
        config = load_json(f"templates/{template_id}.json")
        return Template(
            asset_scanner=AssetScanner(config.asset_filters),
            identity_layer=IdentityLayer(
                clusters=config.identity_clusters,
                profiles=config.subtitle_profiles
            ),
            render_engine=RenderEngine(config.effect_presets),
            timing=TimingConfig(config.timing),
            uploader=Uploader(config.upload_defaults)
        )

    def detect_content_type(self, raw_video: Path) -> str:
        # Auto-detect based on motion patterns, audio profile, metadata
        motion_score = analyze_motion_sample(raw_video)
        if motion_score > 0.6 and has_impact_audio(raw_video):
            return "gaming"
        elif motion_score < 0.2 and has_ambient_audio(raw_video):
            return "travel"
        else:
            return "generic"
```

### 2.3 Runtime Template Selection

```python
# Pseudocode
def select_template(raw_video: Path, user_override: str = None) -> Template:
    if user_override:
        return template_loader.load(user_override)

    detected = template_loader.detect_content_type(raw_video)
    return template_loader.load(f"{detected}_v1")
```

---

## 3. State Machines

### 3.1 Pipeline State Machine

```
[INIT] ──▶ [TEMPLATE_LOAD] ──▶ [ASSET_SELECT] ──▶ [EXTRACT] ──▶ [ANALYZE]
                                                                   │
                                                                   ▼
[UPLOAD] ◀── [FINALIZE] ◀── [MUX] ◀── [RENDER] ◀── [BUILD_TIMELINE]
   │
   └──▶ [CLEANUP] ──▶ [DONE]
```

**Terminal states**: DONE, FATAL (with lock release via atexit)

### 3.2 Asset Cycle State Machine

```
[UNUSED_POOL] ──▶ [SELECT] ──▶ [USED_LIST]
      │                            │
      │◀───────────────────────────┘ (when pool exhausted)
      │
      └──▶ [RESET] ──▶ [SHUFFLE] ──▶ [UNUSED_POOL]
```

Tracks: video_cycle, audio_cycle, video_impacts, audio_peaks, video_segments, audio_segments

### 3.3 Render Frame State Machine

Per-frame decisions based on `output_time` and **template effect presets**:

```
IF template.effects.progressive_zoom AND output_time < hook_time:
  → progressive_zoom_in
ELIF template.effects.impact_flash AND output_time == hook_time:
  → impact_flash
ELIF template.effects.shake AND active_events:
  → shake

IF template.effects.subtitle_burn AND output_time in reseed_segment:
  → inject_reseed_text

IF output_time < template.timing.opening_text_end:
  → render_opening_text

IF template.effects.dark_fade AND output_time >= template.timing.fade_start:
  → dark_fade + ending_text

IF subtitle_active AND output_time >= template.timing.fade_start:
  → render_subtitle
```

---

## 4. Scoring System

### 4.1 Visual Confidence Adjustment

Not a gate — an **adjustment layer**:

```
relevance = visual_strength×0.50 + motion_variation×0.20 + peak_contrast×0.20 + clarity×0.10

IF relevance < template.thresholds.visual_relevance:
  final_score *= template.penalties.low_relevance      (default: 0.60)
ELSE:
  final_score *= template.boosts.high_relevance        (default: 0.92 + relevance×0.22)
```

Template can override `thresholds.visual_relevance`, `penalties.low_relevance`, and `boosts.high_relevance`.

### 4.2 Pool Variance Factor

When all candidates are similar (low variance), scores collapse toward neutral instead of edges:

```
confidence = (range_strength×0.55) + (variance_strength×0.45)
normalized = neutral×(1-confidence) + raw_norm×confidence
```

Keeps scoring meaningful even in homogeneous pools.

---

## 5. Memory & Caching

### 5.1 Analysis Cache

- Audio decode: `voiden_analysis_<hash>.wav` in temp dir
- Video metrics: computed per sampling plan, cached by `(start, duration, frame_skip, offset)`
- Font objects: cached by `(script, size)`
- Template configs: cached in memory after first load

### 5.2 Tracker Persistence

| File | Data | TTL |
|------|------|-----|
| `asset_tracker.json` | Usage counts, last_used | Persistent |
| `asset_cycle.json` | Cycle state, impact/peak history | Persistent |
| `scene_history.json` | Scene bucket usage (20 buckets) | Persistent |
| `uploaded_db.json` | Hash → platform mapping | Persistent |
| `onset_tracker.json` | Onset detection history | Persistent |
| `template_cache.json` | Loaded template configs | Persistent |

### 5.3 In-Memory Caches

```python
_FONT_PATH_CACHE = {}      # script → font path
_FONT_OBJECT_CACHE = {}    # (script, size) → ImageFont
_CACHE = None              # asset_tracker mtime cache
_PAIR_TRACKER = set()     # session-level pair dedup
_TEMPLATE_CACHE = {}       # template_id → Template object
```

---

## 6. Concurrency & Safety

### 6.1 Single-Instance Lock

```python
LOCK_FILE = "voiden.lock"
# PID-based, auto-release via atexit
```

### 6.2 CPU Guard

```python
if cpu > 95%: sleep(3)
elif cpu > MAX_CPU_PERCENT: sleep(1)
```

### 6.3 Subprocess Priority

- Main process: Below normal (Windows) / nice 10 (Unix)
- FFmpeg subprocess: Same priority via creationflags/preexec_fn

---

## 7. Extensibility Points

### 7.1 Adding a New Content Type

1. Create `templates/<type>_v1.json` with all required fields
2. Implement `detect_content_type()` logic in `template_loader.py`
3. Add test assets to `videos/<type>/` and `music/<type>/`
4. Update README roadmap table
5. Run validation: `python -m engine.validate_template <type>_v1`

### 7.2 Adding a New Platform

1. Add to `SUPPORTED_TARGETS` in `uploader.py`
2. Implement `upload_to_<platform>()` function
3. Add config section in `platform_upload_config.json`
4. Update `upload_to_platform()` dispatcher
5. Add template upload_defaults for new platform

### 7.3 Adding a New Effect

1. Add parameter to `config.py` (default value)
2. Add toggle in template JSON under `effect_presets`
3. Implement in `video_processor.py` frame loop with template check
4. Add to `effect_plan` contract validation
5. Update `identity_layer` if style-dependent
6. Update template documentation

---

## 8. Testing Strategy

### 8.1 Unit Tests

- `reseed_scheduler.py`: Self-contained `__main__` block with assertions
- `contract.py`: Schema validation for effect_plan and template JSON
- `timing.py`: FPS snapping correctness
- `template_loader.py`: Template validation and default fallback

### 8.2 Integration Tests

- Full pipeline with known assets → verify output duration
- Template switching: gaming → travel → verify different effects
- Upload with mock API responses → verify retry logic
- Cycle exhaustion → verify reset behavior
- Template auto-detection → verify correct template selected

### 8.3 Performance Tests

- Memory profiling during render
- CPU throttling under load
- Disk space guard triggering
- Template load time (should be < 10ms)

---

## 9. Known Limitations

| Limitation | Mitigation | Template Impact |
|-----------|-----------|-----------------|
| Single-threaded render | Throttled to prevent thermal throttling | All templates |
| No GPU acceleration for analysis | OpenCV optical flow is CPU-only | All templates |
| Fixed output resolution (1080×1920) | Hardcoded in config | All templates |
| Facebook story requires 60s limit | Validated before upload | All templates |
| No real-time preview | Batch processing only | All templates |
| Gaming template only | Travel/Sports/Music in development | Future templates |

---

## 10. Future Roadmap

| Priority | Feature | Template |
|----------|---------|----------|
| P0 | Template validation CLI | All |
| P0 | Auto content-type detection | All |
| P1 | Travel vlog template | Travel |
| P1 | Sports template | Sports |
| P1 | Multi-resolution output (720p, 4K) | All |
| P2 | GPU-accelerated optical flow (CUDA) | All |
| P2 | Music video template | Music |
| P2 | Real-time preview window | All |
| P3 | Fashion/lifestyle template | Fashion |
| P3 | Plugin system for custom effects | All |
| P3 | Web dashboard for template selection | All |
| P3 | Distributed rendering across machines | All |
