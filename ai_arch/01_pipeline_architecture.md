# 1. Pipeline Architecture (in depth)

This document explains how the AI video-analysis subsystem works end to end,
from an instructor pressing **Analyze** to a finished report with per-student
evidence images. It reflects the code **after** the 2026-06-16 fixes; where a
behaviour changed, the change is noted and cross-referenced to
[03_changes_made.md](03_changes_made.md).

---

## 1.1 Where the AI sits in the system

```
Frontend (React/Vite)                Backend (Django + DRF)                 AI package (apis/ai)
─────────────────────                ──────────────────────                 ────────────────────
POST /videos/<id>/analyze/  ──────▶  views.AnalyzeVideoView
                                      └─ services.enqueue_analysis()
                                          └─ django-rq "default" queue
                                              └─ tasks.run_analysis(job_id)        (worker / eager)
                                                  └─ services.build_ai_report(session, job)
                                                      ├─ apis.ai.analyze_video(path) ───▶  THE PIPELINE
                                                      ├─ persist Alert rows
                                                      ├─ services._attach_alert_evidence()
                                                      └─ persist Report row
GET /sessions/<id>/report/  ◀──────  views.SessionReportView (groups Alerts by person)
```

* **`apis/ai/`** is a self-contained package that knows nothing about Django.
  Its single public entry point is `analyze_video(video_path) -> AnalysisResult`.
  Heavy dependencies (`ultralytics`, `torch`, `opencv`) are imported lazily, so
  importing the package is cheap and unit tests can run without model weights.
* **`apis/services.py`** is the Django-aware adapter. `build_ai_report()` calls
  `analyze_video()`, maps the returned events onto `Alert` rows, attaches visual
  evidence, and writes the aggregate `Report`.
* **`apis/tasks.py`** owns the job lifecycle (`QUEUED → PROCESSING → COMPLETED /
  FAILED`) on the django-rq queue. In dev/test the queue runs *eagerly*
  (in-process) so no Redis/worker is needed.

The boundary matters: the AI package is pure and testable; everything
Django-specific (models, media paths, transactions) stays in `services.py`.

---

## 1.2 The AI package — module map

| Module | Responsibility |
|--------|----------------|
| `config.py` | All tunables, resolved from environment variables with defaults. The COCO-class → behaviour map. The looking-away master switch. |
| `frame_extractor.py` | Open the video, expose metadata (fps/size/frames), and **sample every Nth frame** (default 15) as BGR numpy arrays. |
| `yolo_detector.py` | `ObjectDetector` — runs `yolo11x.pt` on a frame and returns kept **phone/laptop** detections (with per-class confidence gating). |
| `pose_analyzer.py` | `PoseAnalyzer` — runs `yolo11x-pose.pt`, estimates head turn from COCO-17 keypoints, returns **looking-away** flags with a person box. |
| `detector.py` | The orchestrator. Runs both detectors over the sampled frames, **consolidates** raw per-frame detections into temporal **events**, and writes the annotated full-length video. |
| `face_tracker.py` | OpenCV Haar-cascade **face tracking** + the **evidence drawing** primitives (boxes, labels, face crops, short clips, H.264 re-encode — GPU **NVENC** when available, else CPU libx264). |
| `model_cache.py` | Process-level cache of loaded YOLO models (each `(weights, device)` loaded once and reused across jobs) + a one-shot **warmup** on first load. A GPU-deploy optimisation — see [07_gpu_deployment.md](07_gpu_deployment.md). |
| `__init__.py` | Exports `analyze_video`, `consolidate_events`, `AnalysisResult`, `AlertEvent`, `config`. |

---

## 1.3 The models

Two pretrained Ultralytics YOLO11-**extra-large** checkpoints:

* **`yolo11x.pt`** — 80-class COCO object detector. Only two classes are kept:
  COCO **67 = cell phone** → `PHONE_DETECTED`, COCO **63 = laptop** → `LAPTOPS`
  (`config.COCO_CLASS_MAP`). Everything else YOLO sees (chairs, books, the wall
  clock, etc.) is discarded by passing `classes=[67, 63]` to `predict()`.
* **`yolo11x-pose.pt`** — person detector + 17 **COCO keypoints** per person
  (nose, eyes, ears, shoulders, elbows, …). Used only for looking-away.

Both are resolved by name; Ultralytics downloads (~220 MB total) and caches them
on first use. They load lazily on first frame and are moved to the resolved
device (GPU `0` or `'cpu'`, see `config.resolve_device()`). On a GPU they run in
**FP16 by default** (`AI_HALF`, applied only when the device is not CPU) and are
loaded **once per process and reused across jobs** via `model_cache.py`, which
also runs a one-shot warmup on first load. See
[07_gpu_deployment.md](07_gpu_deployment.md) for the GPU story end to end.

**COCO-17 keypoint indices used by the heuristic** (`pose_analyzer.py`):

```
0 nose   1 left-eye   2 right-eye   3 left-ear   4 right-ear   5 left-shoulder   6 right-shoulder
```

---

## 1.4 End-to-end data flow

```
                         analyze_video(video_path)
                                   │
            ┌──────────────────────┴───────────────────────┐
            │  _collect_frame_detections()                  │
            │                                               │
   frame_extractor.iter_sampled_frames(every 15th frame)    │
            │                                               │
   for each sampled frame:                                  │
       ObjectDetector.detect(frame)  → [Detection(phone/laptop, conf, bbox)]
       if config.ENABLE_LOOKING_AWAY:                       │   (gated, default OFF)
           PoseAnalyzer.analyze(frame) → [PoseDetection(looking_away, conf, person_bbox)]
            │                                               │
            ▼                                               │
   list[FrameDetection]  (behavior_type, confidence, bbox, frame_number, timestamp_sec)
            │                                               │
            ▼                                               │
   consolidate_events()  ── temporal + per-person tracking ─┘
            │
            ▼
   list[AlertEvent]  (behavior_type, confidence, start_sec, end_sec,
                      frame_count, bbox, bbox_timestamp_sec)
            │
            ├──▶ _write_annotated_video()  → <name>_annotated.mp4   (boxes on the full video)
            │
            ▼
   AnalysisResult(events, annotated_video_path, metadata)
            │
            ▼  (back in services.build_ai_report)
   for each event:  _build_alert()  → Alert(timestamp, type, severity, conf,
                                            metadata{detection_bbox=[x,y,w,h], ...})
            │
            ▼
   _attach_alert_evidence()  → per-alert snapshot_url, clip_url, crop_url, person_id
            │
            ▼
   Report(overall_cheating_probability, total_alerts, alerts_by_type, summary)
```

### Stage A — frame sampling (`frame_extractor.py`)

`iter_sampled_frames(path, sample_every_n=15)` decodes the video with OpenCV and
yields every 15th frame as a `FrameSample(frame, frame_number, timestamp_sec)`.
At ~24–30 fps that is roughly two analysed frames per second — enough to catch a
sustained behaviour, cheap enough to be tractable on long recordings. The stride
is the single biggest lever on runtime.

### Stage B — per-frame detection

**Object detection** (`ObjectDetector.detect`):

```python
predict_kwargs = {
    'conf': self.confidence,          # coarse floor (0.30)
    'classes': [67, 63],              # phone, laptop only
    'imgsz': self.imgsz,              # 960  (was the 640 default — the phone-recall fix)
    'agnostic_nms': self.agnostic_nms,
    'verbose': False,
    'device': self.device,
}
if self.half and self.device != 'cpu':
    predict_kwargs['half'] = True     # FP16 only on GPU
results = self.model(frame, **predict_kwargs)
# ... for each box:
keep_threshold = self.class_confidence.get(behavior_type, self.confidence)  # phone 0.30, laptop 0.85
if confidence < keep_threshold:
    continue                          # <-- per-class gate: kills paper-as-laptop FPs
```

The detector returns `Detection(behavior_type, confidence, bbox=(x1,y1,x2,y2))`.
The bbox is in **original source-frame pixel coordinates** — Ultralytics rescales
boxes back to the input frame size regardless of `imgsz`, an invariant that the
whole overlay fix depends on (verified empirically; see
[05_verification.md](05_verification.md)).

**Pose / looking-away** (`PoseAnalyzer.analyze`) — only runs when
`config.ENABLE_LOOKING_AWAY` is true. It produces a `PoseDetection(LOOKING_AWAY,
confidence, bbox=person_box)` per flagged person. The decision logic is in §1.6.

### Stage C — temporal consolidation (`consolidate_events`)

Raw per-frame detections are noisy: the same phone appears in many sampled
frames, two students might both look away, a flicker might appear for one frame.
`consolidate_events()` turns the raw `FrameDetection` list into clean
`AlertEvent`s using **two-axis tracking**:

1. **Group by behaviour type.**
2. Within a type, maintain **open tracks**. A new detection extends an existing
   track only if it is (a) within `MERGE_WINDOW_SEC` (5 s) in time **and**
   (b) spatially close to the track's last centre — `_same_track()` compares
   bbox centres with a distance threshold that scales with box size, so two
   students (or two phones) become two separate events instead of merging.
3. Otherwise it opens a **new event**.

Each surviving event must then clear **noise gates**:

* **Direct-evidence objects** (`PHONE_DETECTED`, `LAPTOPS`) **bypass** the
  duration gate — even a single sighting of a forbidden object is a real
  violation.
* **Looking-away** must last ≥ `MIN_EVENT_DURATION_SEC` (2 s) **and** span
  ≥ `LOOKING_AWAY_MIN_CONSECUTIVE` (3) consecutive sampled frames, so a single
  momentary glance never alerts.

**What the fix added here:** each `AlertEvent` now carries `bbox` and
`bbox_timestamp_sec` — the box of the **strongest-confidence** frame in the
track, plus the timestamp of that frame. On track-open the first detection's box
is taken; on extend, the box is replaced only when a later frame has *higher*
confidence **and** a usable (non-degenerate) box. This is the data that lets the
report draw the real detection (previously the box was thrown away here).

### Stage D — the annotated full-length video (`_write_annotated_video`)

Independently of the report evidence, the pipeline can write a
`<name>_annotated.mp4`: it re-reads every frame, draws the surviving detections'
boxes on the frames that back an event, overlays an `ALERT: ...` banner during
each event window, and re-encodes to browser-playable H.264. The re-encode uses
the GPU's hardware encoder **NVENC** (`h264_nvenc`) when an NVENC-capable ffmpeg
is present (`AI_NVENC=auto`), falling back to CPU `libx264` otherwise — see
[07_gpu_deployment.md](07_gpu_deployment.md) §7.9. This path always used the real
detection box; it was **not** the source of the wrong-box bug.

---

## 1.5 From events to a report (`services.py`)

### `_build_alert(session, event)` — carry the box into the DB

For each consolidated event it constructs an unsaved `Alert`:

* `timestamp_sec` = the strongest-evidence frame's timestamp (so the report
  caption and the drawn frame coincide), falling back to the event start.
* `severity` = `_severity_for(confidence)` (HIGH ≥ 0.8, MEDIUM ≥ 0.5, else LOW).
* `metadata` carries `source`, `start_sec`, `end_sec`, `duration_sec`,
  `frame_count`, and — the key addition —
  **`detection_bbox = [x, y, w, h]`** (original-frame pixels) and
  **`detection_timestamp_sec`**. `getattr(event, 'bbox', None)` is used so the
  mocked test event (a `SimpleNamespace` with no bbox) keeps working.

### `_attach_alert_evidence(video, session, alerts)` — the overlay

This is where the user-visible evidence images are produced. Per alert it writes
three artifacts under `MEDIA_ROOT`:

| Artifact | Field | What it shows |
|----------|-------|---------------|
| **Annotated frame** | `alert.snapshot_url` | The full frame with a **green box on the real detection** + `Person N - <Behaviour>` label, and gray boxes on other tracked people. |
| **Clip** | `alert.clip_url` | A 3-second H.264 clip starting at the detection, the same overlay baked onto every frame. |
| **Face crop** | `metadata['crop_url']` | The flagged person's **face** only (avatar in the report header). |

The logic (post-fix):

1. Read the carried `detection_bbox` from the alert metadata and convert
   `[x,y,w,h] → (x1,y1,x2,y2)`. Read `detection_timestamp_sec` → `render_ts`.
2. Run the **Haar face tracker once** (`track_persons`) and query it at
   `render_ts` for the nearest face (`face_bbox`) and all faces in frame
   (`others`). Haar is now used **only** for: the avatar crop, the gray context
   boxes, and **spatial** person attribution.
3. **Attribution:** `_nearest_person(det_xyxy, faces)` picks the tracked face
   whose centre is closest to the detection box → `person_id`. (Previously the
   code used the *time*-nearest face, which had no spatial link to the object.)
4. **Draw:** the green flagged box = the **detection box**; gray boxes = the
   other faces; the avatar crop = the **associated face box** (never the object
   box, so a phone alert's avatar is still a face).
5. If no carried box exists (e.g. the mocked test path), it falls back to the
   previous Haar-face behaviour exactly — so unit tests are unaffected.

### `build_ai_report` — the aggregate

After persisting alerts and evidence it writes a `Report`: an overall
cheating probability (mean severity weight, capped at 1.0), `total_alerts`,
`alerts_by_type`, `person_count` (distinct `person_id`s), and a text summary.

---

## 1.6 The looking-away heuristic (v2)

Looking-away has **no ground-truth label model**; it is a geometric heuristic on
2D keypoints. The v2 design (replacing the brittle nose-vs-eye-midpoint yaw)
combines up to three votes and flags `LOOKING_AWAY` only when at least
`LOOKING_AWAY_MIN_VOTES` (2) fire **and the mandatory V2 vote is among them**:

| Vote | Signal | Why |
|------|--------|-----|
| **V2 (mandatory)** | `|nose_x − shoulder_mid_x| / shoulder_width > 0.35` | Pure **horizontal** offset normalised by shoulder width → scale-invariant and reads **yaw**. A head bowed straight **down** to write keeps the nose centred, so V2 stays low → **looking down can never flag**. |
| **V1** | ear-visibility asymmetry (one ear confidently visible, the other hidden, or a large confidence ratio) | In a profile the far ear disappears; the visible side also gives turn **direction**. |
| **V3** | offset `≥ 0.55` (deep) | A horizontal offset large enough to be self-corroborating. |

Confidence ramps `0→1` from the "sideways" (0.35) to the "turned-back" (0.55)
ratio. The **person box** is taken from the pose detector's box, or — when that
is missing — derived from the bounding extent of the confident keypoints
(never `(0,0,0,0)`, which previously collapsed the per-person tracking and
merged two students into one event).

**Why V2 is mandatory:** an earlier design let ear-asymmetry (V1) flag on its
own, but a student bowed over their paper often *occludes one ear* with zero
yaw — V1 alone would false-flag looking-down. Requiring V2 closes that hole.

> **Critical caveat:** this heuristic is correct for a **frontal** camera but
> **not** for the oblique/overhead camera in the test footage, where perspective
> trips V2 and V1 for forward-facing students. That is why looking-away is
> **OFF by default** (`config.ENABLE_LOOKING_AWAY`). See
> [02_investigation_findings.md](02_investigation_findings.md) §2.5 and
> [06_conclusion_and_followups.md](06_conclusion_and_followups.md).

---

## 1.7 Contracts (do not break these)

| Contract | Where | Note |
|----------|-------|------|
| `AnalysisResult(events, annotated_video_path, metadata)` | `apis/ai` ↔ `services` | Duck-typed by the test mock `fake_analysis_result()`. |
| `AlertEvent` fields | `apis/ai` ↔ `services` | `bbox`/`bbox_timestamp_sec` are **optional with defaults**, read via `getattr`, so the `SimpleNamespace` mock (no bbox) still works. |
| `Alert.metadata` keys | `services` → DB → API → frontend | `person_id`, `crop_url`, `bbox`, and the new `detection_bbox=[x,y,w,h]`, `detection_timestamp_sec`. The serializer passes `metadata` through verbatim — no serializer change needed. |
| `metadata['bbox'] = [x, y, w, h]` | frontend `videos.ts` | The frontend declares it but **does not draw it** — it renders the baked `snapshot_url`/`clip_url`/`crop_url`. So the box shape change is display-safe. |
| `person_id` grouping | `views.SessionReportView` | Alerts are grouped into "Person N" sections by `metadata['person_id']`; the avatar is the highest-confidence `crop_url`. Spatial attribution keeps this meaningful. |
| Pixel coordinate space | everywhere | All boxes are **original source-frame xyxy**, regardless of `imgsz`. The overlay draws on the full-res frame. |

See [03_changes_made.md](03_changes_made.md) for the exact edits and
[04_configuration_reference.md](04_configuration_reference.md) for every tunable.
