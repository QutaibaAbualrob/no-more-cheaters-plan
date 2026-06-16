# 3. Changes Made (file by file)

Six source files in `no-more-cheaters-backend/` plus the standalone tester.
Every change is environment-configurable and preserves the existing DB / API /
frontend contracts and the mocked test suite (58/58 passing). Line numbers are
approximate (as of the documented revision); the function/region names are
authoritative.

---

## 3.1 `apis/ai/config.py` — tunables, class thresholds, looking-away gate

**Added** `_env_bool(name, default)` (mirrors `_env_float`/`_env_int`; accepts
`1/true/yes/on` and `0/false/no/off`).

**Added inference-resolution block:**

```python
OBJECT_IMGSZ = max(32, _env_int('AI_OBJECT_IMGSZ', 960))
POSE_IMGSZ   = max(32, _env_int('AI_POSE_IMGSZ', 960))
OBJECT_HALF  = _env_bool('AI_OBJECT_HALF', False)        # FP16, GPU-only, opt-in
OBJECT_AGNOSTIC_NMS = _env_bool('AI_AGNOSTIC_NMS', False)  # near-inert; opt-in
```

**Changed confidence model** from one global threshold to a coarse floor + a
**per-class keep threshold**:

```python
OBJECT_CONFIDENCE = _env_float('AI_OBJECT_CONFIDENCE', 0.30)   # was 0.40 — the predict() floor
CLASS_CONFIDENCE = {
    PHONE_DETECTED: _env_float('AI_PHONE_CONFIDENCE', 0.30),   # phones: the real target
    LAPTOPS:        _env_float('AI_LAPTOP_CONFIDENCE', 0.85),  # paper FPs reach 0.79 -> 0.85 rejects them
}
```

**Added the v2 looking-away constants** (ear visibility/asymmetry, the
shoulder-normalised nose-offset thresholds, the vote count) and the
**master switch**:

```python
EAR_VISIBLE_CONF = 0.60 ; EAR_HIDDEN_CONF = 0.35 ; EAR_CONF_RATIO = 2.5
NOSE_SHOULDER_SIDEWAYS = 0.35 ; NOSE_SHOULDER_TURNED = 0.55
LOOKING_AWAY_MIN_VOTES = 2
ENABLE_LOOKING_AWAY = _env_bool('AI_ENABLE_LOOKING_AWAY', False)   # DEFAULT OFF
```

The old `LOOKING_AWAY_ANGLE_DEG` / `NOSE_DEPTH_RATIO` / `LOOKING_AWAY_RATIO`
constants are **kept** (unused by v2) for back-compat with the
`PoseAnalyzer.__init__` defaults and the standalone harness.

---

## 3.2 `apis/ai/yolo_detector.py` — resolution + per-class gating

**`ObjectDetector.__init__`** gains `imgsz`, `half`, `agnostic_nms`,
`class_confidence` (defaulting to the config values; `class_confidence` stored as
a dict copy).

**`ObjectDetector.detect`** now passes `imgsz` and `agnostic_nms`, adds `half`
**only when the device is not CPU** (FP16 errors on CPU; and note device `0` is
falsy so the guard is `self.device != 'cpu'`, not `if self.half and self.device`),
and applies the per-class keep threshold **after** inference:

```python
keep_threshold = self.class_confidence.get(behavior_type, self.confidence)
if confidence < keep_threshold:
    continue   # phone kept >= 0.30, laptop kept >= 0.85
```

A comment records the load-bearing invariant: **Ultralytics rescales `box.xyxy`
back to original source-frame pixels regardless of `imgsz`**, so the bbox is
directly drawable on the full-resolution frame downstream.

---

## 3.3 `apis/ai/detector.py` — carry the box, gate the pose pass

**`AlertEvent`** gains two optional fields (after `frame_count`, with defaults so
positional construction and the `SimpleNamespace` test mock still work):

```python
bbox: tuple | None = None              # strongest-evidence detection box (xyxy, original frame)
bbox_timestamp_sec: float | None = None
```

**`consolidate_events`** now carries the box:

* On **track-open**, the new `AlertEvent(...)` is built with `bbox=det.bbox` and
  `bbox_timestamp_sec=det.timestamp_sec`.
* On **track-extend**, the box is replaced only when a later frame is **stronger
  and usable** (guarded so a higher-confidence-but-degenerate pose box never
  overwrites a good earlier box):

  ```python
  if det.confidence > event.confidence and center is not None:
      event.bbox = det.bbox
      event.bbox_timestamp_sec = det.timestamp_sec
  ```

  The existing per-person tracking (`_bbox_center`, `_same_track`, the
  `match['center']` update) is **untouched**, so two students / two phones still
  separate into distinct events.

**`_collect_frame_detections`** gates the pose pass behind the master switch (so
disabled looking-away also costs zero compute):

```python
if config.ENABLE_LOOKING_AWAY:
    for pose in pose_analyzer.analyze(sample.frame):
        detections.append(FrameDetection(...))
```

---

## 3.4 `apis/ai/pose_analyzer.py` — v2 looking-away + a real person box

* Added COCO keypoint constants `_LEFT_EAR=3`, `_RIGHT_EAR=4`,
  `_LEFT_SHOULDER=5`, `_RIGHT_SHOULDER=6`.
* `__init__` accepts the new config thresholds (`ear_visible_conf`,
  `ear_hidden_conf`, `ear_conf_ratio`, `nose_shoulder_sideways`,
  `nose_shoulder_turned`, `min_votes`, `imgsz`); old `angle_threshold_deg` /
  `nose_depth_ratio` kept but unused.
* `analyze()` rewritten as the **3-vote heuristic** (§1.6): mandatory horizontal
  nose-vs-shoulder offset (V2), ear asymmetry (V1), deep-offset corroboration
  (V3); flag only when `votes >= min_votes` **and V2 fired**. Per-keypoint
  confidence is read from `keypoints.conf`, falling back to the packed
  `keypoints.data[..., 2]` tensor when `conf` is `None`. The flag confidence
  ramps `0→1` from the sideways to the turned ratio.
* Added **`_person_bbox()`**: returns the detector's person box when present,
  else the **bounding extent of the confident keypoints** — never `(0,0,0,0)`.
  This is the fix for audit item **H11** (a degenerate box made `_bbox_center`
  return `None`, which merged two looking-away students into one event).
* Removed the now-unused `import math`; updated the module docstring.

---

## 3.5 `apis/ai/face_tracker.py` — the `???` fix

The drawn flagged label `f'{label} — {behavior_label}'` used an **em-dash**,
which OpenCV's Hershey font renders as `???`. Replaced `—` with `-` in all three
label sites (`_draw_person_boxes`, the `extract_annotated_frame` fallback, the
`extract_clip` fallback). The em-dashes in *comments/docstrings* were left
(source only, never drawn). No signature changes.

---

## 3.6 `apis/services.py` — draw the REAL box; spatial attribution

This is the heart of the overlay fix. Three pieces were added/rewritten.

**`_build_alert(session, event)`** (new helper, replaces the inline Alert
list-comprehension in `build_ai_report`): copies the temporal metadata and adds

```python
metadata['detection_bbox'] = [x, y, w, h]          # from event.bbox (xyxy -> xywh)
metadata['detection_timestamp_sec'] = <peak frame> # the strongest-evidence frame
timestamp_sec = int(round(detection_timestamp))    # so caption + drawn frame coincide
```

It uses `getattr(event, 'bbox', None)` and validates `w>0, h>0`, so a `None`/
degenerate box (or the mock event) simply omits `detection_bbox`.

**`_box_center(box)` / `_nearest_person(det_box, candidates)`** (new helpers):
attribute a detection to the tracked face whose centre is **spatially nearest**
the detection box — replacing the old *time*-nearest guess that had no spatial
relationship to the object.

**`_attach_alert_evidence(...)`** rewritten:

* Read `detection_bbox` → `det_xyxy`; read `detection_timestamp_sec` →
  `render_ts`. Run `track_persons` **once**, query it at `render_ts`.
* If a carried box exists:
  * attribute `person_id` via `_nearest_person`;
  * the **drawn green box = the detection box** (it takes the flagged person's
    slot in the boxes list); other faces are gray;
  * the **avatar crop uses the associated *face* box** — never the object box,
    so a phone/laptop alert's avatar is still a face;
  * `render_ts` is used for the snapshot/clip/crop so the drawn box and the
    frame coincide (the strongest-evidence frame).
* If **no** carried box exists (mocked tests / a None-bbox event): the code
  falls back to the **previous** Haar-face / centred-fallback behaviour exactly,
  so the unit suite is unaffected.

`build_ai_report` now builds alerts via `[_build_alert(session, e) for e in
result.events]`.

---

## 3.7 `test-photos/run.py` — the standalone tester, aligned to production

* Added `CLASS_FLOORS = {67: 0.30, 63: 0.85}` and applied them so paper
  "laptops" drop out exactly as in the pipeline.
* Replaced the absolute-pixel pose thresholds (40 px / 80 px, tied to one camera
  distance) with the **shoulder-width-normalised ratios** (0.35 / 0.55) used by
  the production heuristic, and added a comment that looking-away is
  frontal-camera-only and OFF by default in the pipeline.
* Kept `IMG_SIZE = 1280`.

---

## 3.8 Design decisions & how they were pressure-tested

The fix was designed against the Ultralytics YOLO11 docs and then run through an
adversarial review. The notable decisions:

| Decision | Why |
|----------|-----|
| **Per-class thresholds** (not one raised global conf) | A single high conf would kill weak true phones along with paper-laptops. Phones need a low bar **and** high resolution; laptops need a high bar. |
| **Laptop = 0.85** (review draft said 0.60) | The 0.60 figure came from a stale "paper FPs are 0.27–0.48" estimate. The measured distribution shows FPs reach **0.79**; 0.85 is required to reject them. |
| **`imgsz` = 960** (review draft said 1280) | 960 is the chosen balance of recall vs ~2–2.5× compute; 1280 is one env-var away for maximum recall. |
| **Render at the peak-confidence frame** | The carried bbox is the strongest frame's box; rendering the snapshot at that same timestamp eliminates the box-vs-frame drift a moving person would otherwise cause. |
| **Avatar always uses the face box** | Feeding the object box into the face crop would make a phone alert's avatar a picture of a phone. |
| **Object alerts attributed *spatially*** | Time-nearest Haar query has no spatial link to the object and would mis-name "Person N". Nearest-face-to-box is meaningful. |
| **Guarded bbox update on extend** | Prevents a later high-confidence but degenerate (collapsed-keypoint) box from overwriting a good earlier box. |
| **V2 (horizontal offset) mandatory** | A bowed-over-paper head can occlude one ear (satisfying V1) with zero yaw; requiring V2 means looking-down can never flag. |
| **`agnostic_nms` default OFF** | With `classes=[63,67]` paper/book (class 73) is filtered *before* NMS, so agnostic NMS cannot merge a paper-vs-laptop overlap — it is near-inert here. Kept as an opt-in flag only. |
| **Looking-away OFF by default** | It cannot be made reliable on an oblique camera with 2D keypoints (Finding 4). Better to ship it opt-in than to flag the whole class. |

See [05_verification.md](05_verification.md) for proof these changes behave as
intended, and [04_configuration_reference.md](04_configuration_reference.md) for
the full knob list.
