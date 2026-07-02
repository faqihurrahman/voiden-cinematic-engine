# SYSTEM_DESIGN.md — Voiden Cinematic Engine

> **Document Version**: 1.0  
> **Engine Version**: V31.3  
> **Last Updated**: 2026-07-02

---

## 1. Design Principles

### 1.1 Scheduler-Based Architecture

The engine rejects heuristic layering in favor of a **central timing authority**. All timing decisions flow from a single source:

- `output_events` → drives render frame decisions
- `segment_timeline` → drives candidate selection
- `effect_plan` → drives audio trimming and time remapping

No parallel logic paths. No competing timing systems.

### 1.2 Determinism

With `SEED_MODE = True`, the entire pipeline produces identical output for identical input:
- Random choices use fixed seed (42)
- Hash-based stable units for identity layer
- Deterministic sampling plans per candidate pair

### 1.3 Minimal Overlap

Each module has exactly one responsibility:

| Module | Responsibility | Never Does |
|--------|---------------|-----------|
| `audio_analyzer` | Peak/impact detection | Render decisions |
| `segment_selector` | Candidate scoring | Timeline building |
| `momentum_engine` | Timeline sequencing | Video analysis |
| `identity_layer` | Style profiling | Timing calculations |
| `video_processor` | Frame rendering | Audio analysis |
| `uploader` | Platform upload | Video processing |

---

## 2. State Machines

### 2.1 Pipeline State Machine

```
[INIT] ──▶ [ASSET_SELECT] ──▶ [EXTRACT] ──▶ [ANALYZE]
                                              │
                                              ▼
[UPLOAD] ◀── [FINALIZE] ◀── [MUX] ◀── [RENDER] ◀── [BUILD_TIMELINE]
   │
   └──▶ [CLEANUP] ──▶ [DONE]
```

**Terminal states**: DONE, FATAL (with lock release via atexit)

### 2.2 Asset Cycle State Machine

```
[UNUSED_POOL] ──▶ [SELECT] ──▶ [USED_LIST]
      │                            │
      │◀───────────────────────────┘ (when pool exhausted)
      │
      └──▶ [RESET] ──▶ [SHUFFLE] ──▶ [UNUSED_POOL]
```

Tracks: video_cycle, audio_cycle, video_impacts, audio_peaks, video_segments, audio_segments

### 2.3 Render Frame State Machine

Per-frame decisions based on `output_time`:

```
IF output_time < hook_time:
  → progressive_zoom_in
ELIF output_time == hook_time:
  → impact_flash + shake
ELIF output_time > hook_time:
  → zoom_recovery (speed from identity_layer)

IF output_time in reseed_segment:
  → inject_reseed_text

IF output_time < 3.5s:
  → render_opening_text

IF output_time >= 75% duration:
  → dark_fade + ending_text

IF subtitle_active AND output_time >= 75%:
  → render_subtitle
```

---

## 3. Scoring System

### 3.1 Visual Confidence Adjustment

Not a gate — an **adjustment layer**:

```
relevance = visual_strength×0.50 + motion_variation×0.20 + peak_contrast×0.20 + clarity×0.10

IF relevance < 0.42:
  final_score *= 0.60      (penalized)
ELSE:
  final_score *= 0.92 + relevance×0.22  (boosted)
```

This prevents false positives without hard rejection.

### 3.2 Pool Variance Factor

When all candidates are similar (low variance), scores collapse toward neutral instead of edges:

```
confidence = (range_strength×0.55) + (variance_strength×0.45)
normalized = neutral×(1-confidence) + raw_norm×confidence
```

Keeps scoring meaningful even in homogeneous pools.

---

## 4. Memory & Caching

### 4.1 Analysis Cache

- Audio decode: `voiden_analysis_<hash>.wav` in temp dir
- Video metrics: computed per sampling plan, cached by `(start, duration, frame_skip, offset)`
- Font objects: cached by `(script, size)`

### 4.2 Tracker Persistence

| File | Data | TTL |
|------|------|-----|
| `asset_tracker.json` | Usage counts, last_used | Persistent |
| `asset_cycle.json` | Cycle state, impact/peak history | Persistent |
| `scene_history.json` | Scene bucket usage (20 buckets) | Persistent |
| `uploaded_db.json` | Hash → platform mapping | Persistent |
| `onset_tracker.json` | Onset detection history | Persistent |

### 4.3 In-Memory Caches

```python
_FONT_PATH_CACHE = {}      # script → font path
_FONT_OBJECT_CACHE = {}    # (script, size) → ImageFont
_CACHE = None              # asset_tracker mtime cache
_PAIR_TRACKER = set()     # session-level pair dedup
```

---

## 5. Concurrency & Safety

### 5.1 Single-Instance Lock

```python
LOCK_FILE = "voiden.lock"
# PID-based, auto-release via atexit
```

### 5.2 CPU Guard

```python
if cpu > 95%: sleep(3)
elif cpu > MAX_CPU_PERCENT: sleep(1)
```

### 5.3 Subprocess Priority

- Main process: Below normal (Windows) / nice 10 (Unix)
- FFmpeg subprocess: Same priority via creationflags/preexec_fn

---

## 6. Extensibility Points

### 6.1 Adding a New Platform

1. Add to `SUPPORTED_TARGETS` in `uploader.py`
2. Implement `upload_to_<platform>()` function
3. Add config section in `platform_upload_config.json`
4. Update `upload_to_platform()` dispatcher

### 6.2 Adding a New Effect

1. Add parameter to `config.py`
2. Implement in `video_processor.py` frame loop
3. Add to `effect_plan` contract validation
4. Update `identity_layer` if style-dependent

### 6.3 Adding a New Cluster

1. Add to `CLUSTERS` tuple in `identity_layer.py`
2. Add token nudges to `_TOKEN_NUDGES`
3. Add weight formula
4. Add cadence pattern
5. Add subtitle presentation profile

---

## 7. Testing Strategy

### 7.1 Unit Tests

- `reseed_scheduler.py`: Self-contained `__main__` block with assertions
- `contract.py`: Schema validation for effect_plan
- `timing.py`: FPS snapping correctness

### 7.2 Integration Tests

- Full pipeline with known assets → verify output duration
- Upload with mock API responses → verify retry logic
- Cycle exhaustion → verify reset behavior

### 7.3 Performance Tests

- Memory profiling during render
- CPU throttling under load
- Disk space guard triggering

---

## 8. Known Limitations

| Limitation | Mitigation |
|-----------|-----------|
| Single-threaded render | Throttled to prevent thermal throttling |
| No GPU acceleration for analysis | OpenCV optical flow is CPU-only |
| Fixed output resolution (1080×1920) | Hardcoded in config |
| Facebook story requires 60s limit | Validated before upload |
| No real-time preview | Batch processing only |

---

## 9. Future Roadmap

| Priority | Feature |
|----------|---------|
| P1 | Multi-resolution output (720p, 4K) |
| P1 | GPU-accelerated optical flow (CUDA) |
| P2 | Real-time preview window |
| P2 | Plugin system for custom effects |
| P3 | Web dashboard for job monitoring |
| P3 | Distributed rendering across machines |
