# 4. Configuration Reference

Every tunable in `apis/ai/config.py` is read from an environment variable with a
default, so behaviour can be changed per-deployment without touching code (e.g.
in the backend `.env`). Defaults below are the production values after the fix.

---

## 4.1 Models & device

| Env var | Default | Meaning |
|---------|---------|---------|
| `AI_OBJECT_MODEL` | `yolo11x.pt` | Object-detection weights. |
| `AI_POSE_MODEL` | `yolo11x-pose.pt` | Pose weights. |
| `AI_DEVICE` | `auto` | `auto` → GPU if available else CPU; `cpu`; `0`/`cuda` → first GPU. Resolved once and cached. |

## 4.2 Inference resolution & speed

| Env var | Default | Meaning |
|---------|---------|---------|
| `AI_OBJECT_IMGSZ` | `960` | Object-detection inference size. **640** = fast but loses small phones; **960** = balanced (≈2–2.5× compute); **1280** = best small-object recall (≈3.5–4×). Box coords are unaffected by this. |
| `AI_POSE_IMGSZ` | `960` | Same for the pose pass (distant-student keypoints). |
| `AI_OBJECT_HALF` | `false` | FP16 inference. **GPU only** — passed to `predict()` only when the device is not CPU. Lowers GPU cost/memory. |
| `AI_AGNOSTIC_NMS` | `false` | Class-agnostic NMS. Near-inert with `classes=[63,67]`; opt-in only. |
| `AI_SAMPLE_RATE` | `15` | Analyse every Nth frame. The biggest single lever on total runtime. Lower = more thorough + slower. |

## 4.3 Detection confidence

| Env var | Default | Meaning |
|---------|---------|---------|
| `AI_OBJECT_CONFIDENCE` | `0.30` | Coarse `predict()` floor — candidates below this are never returned. |
| `AI_PHONE_CONFIDENCE` | `0.30` | Per-class **keep** threshold for phones. Equal to the floor (phones are the target; recall comes from `imgsz`). |
| `AI_LAPTOP_CONFIDENCE` | `0.85` | Per-class **keep** threshold for laptops. High because exam paper false-positives reach ~0.79. Lower toward 0.5 **only** if your exams genuinely use laptops and real ones are missed. |

> The per-class keep thresholds are applied **after** inference because
> Ultralytics `predict()` accepts only one global `conf`.

## 4.4 Looking-away (head-pose) — OFF by default

| Env var | Default | Meaning |
|---------|---------|---------|
| `AI_ENABLE_LOOKING_AWAY` | `false` | **Master switch.** Off because the 2D heuristic is unreliable on oblique/overhead cameras (see findings). Turn on **only** for a roughly frontal, student-facing camera. |
| `AI_NOSE_SHOULDER_SIDEWAYS` | `0.35` | V2 flag threshold: `|nose_x − shoulder_mid_x| / shoulder_width`. |
| `AI_NOSE_SHOULDER_TURNED` | `0.55` | V2 full-confidence (deep-turn) threshold; also the V3 self-corroboration trigger. |
| `AI_EAR_VISIBLE_CONF` | `0.60` | An ear counts as "seen" above this confidence. |
| `AI_EAR_HIDDEN_CONF` | `0.35` | An ear counts as "hidden" below this confidence. |
| `AI_EAR_CONF_RATIO` | `2.5` | One ear's confidence ÷ the other's, above which V1 fires. |
| `AI_LOOKING_AWAY_VOTES` | `2` | Votes required to flag (V2 is **always additionally** required). |
| `AI_POSE_CONFIDENCE` | `0.50` | Minimum per-keypoint confidence before a landmark is trusted. |
| `AI_CONSECUTIVE_DETECTIONS` | `3` | A looking-away event must span ≥ this many **consecutive** sampled frames. |
| `AI_HEAD_TURN_ANGLE_DEG`, `AI_NOSE_DEPTH_RATIO`, `AI_LOOKING_AWAY_RATIO` | 45.0 / 0.55 / 0.35 | **Legacy** (old v1 heuristic); retained for back-compat, unused by v2. |

## 4.5 Temporal consolidation

| Env var | Default | Meaning |
|---------|---------|---------|
| `AI_MERGE_WINDOW_SEC` | `5.0` | Same-class detections within this window (and spatially close) merge into one event. |
| `AI_MIN_EVENT_SEC` | `2.0` | Looking-away events shorter than this are dropped as noise. (Phone/laptop bypass this gate.) |

---

## 4.6 Recommended profiles

**Default (this project / oblique room camera, paper-based exam):**

```
# detection only; looking-away off
AI_OBJECT_IMGSZ=960
AI_PHONE_CONFIDENCE=0.30
AI_LAPTOP_CONFIDENCE=0.85
AI_ENABLE_LOOKING_AWAY=false
```

**Maximum phone recall (GPU, you accept slower runs):**

```
AI_OBJECT_IMGSZ=1280
AI_OBJECT_HALF=true        # GPU only
AI_PHONE_CONFIDENCE=0.25   # accept more phone candidates (still violations)
AI_LAPTOP_CONFIDENCE=0.85  # keep the laptop bar high — 1280 multiplies paper FPs
```

**Frontal proctoring camera, want looking-away:**

```
AI_ENABLE_LOOKING_AWAY=true
# then recalibrate on a few labelled clips from THAT camera:
AI_NOSE_SHOULDER_SIDEWAYS=0.35   # raise if forward students still flag
AI_NOSE_SHOULDER_TURNED=0.55
AI_LOOKING_AWAY_VOTES=2
AI_CONSECUTIVE_DETECTIONS=3      # raise to require a longer sustained turn
```

**Laptops genuinely allowed/used in the exam (so a laptop is normal, not a
violation):** remove `63: LAPTOPS` from `COCO_CLASS_MAP`, or simply do not act on
`LAPTOPS` alerts downstream. Conversely, if laptops are forbidden and present,
lower `AI_LAPTOP_CONFIDENCE` and expect some paper false-positives.

---

## 4.7 Always tune, then verify

Whenever you change a threshold, re-run the standalone tester on a representative
frame and confirm the result visually:

```
# from no-more-cheaters-backend/
python test-photos/run.py <frame_or_dir>
```

For per-camera looking-away calibration, capture a clip where a student
**clearly** turns to a neighbour and another where students only **write**, and
adjust until the first flags and the second does not.
