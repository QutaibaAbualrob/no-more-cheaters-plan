# 2. Investigation & Findings

This is the evidence trail. Every claim below comes from running the actual
models on the actual recording (`school.mp4`), not from reading code alone.

---

## 2.1 Method

* **Subject video:** `media/exam-videos/<session>/school.mp4` — a documentary
  classroom clip, **1280×720**, **23.976 fps**, **~2314 s** total. It is a cut
  montage: the populated classroom scene runs roughly **t = 15 s … 180 s**; the
  opening seconds are an outdoor face close-up (which is why the original
  `Person 6 — frame at 7s` evidence looked like it came from "nowhere": at 7 s
  the video is not the classroom at all).
* **Tools:** `yolo11x.pt` (object) and `yolo11x-pose.pt` (pose) via Ultralytics,
  on the project's `.venv`, scripted to extract frames at chosen timestamps,
  run inference at varying `imgsz`/`conf`, dump every detection with class +
  confidence + bbox, and render annotated frames for visual confirmation.
* **Proof images** were written to `no-more-cheaters-backend/diag_out/`
  (untracked). Key ones referenced below: `class_objboxes_123s.jpg`,
  `class_pose_123s.jpg`, `signals_123s.jpg`, `e2e_snapshot.jpg`.

---

## 2.2 Finding 1 — the "laptops" are exam paper

Rendering YOLO's **object** boxes (classes phone+laptop) on the classroom frame
at **t = 123 s** produced **six "laptop" boxes** at confidence **0.27–0.48** —
and every single one sat on a **sheet of white exam paper or a book** on a desk.
The frame at t = 45 s showed three more, all paper. There is **no laptop in the
room**. A bright rectangular sheet on a desk looks like an open laptop to the
detector.

Because the old global threshold was `0.40`, the detections at 0.43/0.48 cleared
it and became real `LAPTOPS` alerts.

### Confidence distribution (the decisive measurement)

Over **55 sampled classroom frames** (t = 15–180 s), counting every phone/laptop
candidate down to conf ≥ 0.05:

| imgsz | LAPTOP (all false positives) | PHONE |
|-------|------------------------------|-------|
| **640** (old default) | n=90, mean 0.13, **max 0.74**; ≥0.40: **3**, ≥0.50: 2, ≥0.60: 2, ≥0.70: 1 | n=3, **max 0.11** |
| **1280** | n=273, mean 0.20, **max 0.79**; ≥0.40: **51**, ≥0.50: 19, ≥0.60: 5, ≥0.70: 2 | n=9, **max 0.22** |

Three things fall out of this table:

1. **Paper false positives are not low-confidence noise** — they reach
   **0.74–0.79**. A threshold of 0.60 (as an early draft proposed) would *not*
   clean them; **0.85** does. This is why the laptop keep-threshold is 0.85.
2. **Raising `imgsz` multiplies laptop FPs ~17×** (3 → 51 above 0.40). So you
   cannot naively raise resolution for phone recall without independently
   raising the laptop bar — the two must move together. This drove the
   **per-class threshold** design.
3. **Phones are essentially invisible in this video** — max 0.22 even at 1280.
   Either there are none, or they are too small/occluded to detect. A correct
   system therefore reports **no phones here** (and certainly should not invent
   them).

The object detector also routinely mislabels other room objects (`microwave`,
`refrigerator`, `dining table`, `tv`, `keyboard`, `book`, `backpack`,
`handbag`, `bottle`, `tie`, `cup`) — irrelevant because only classes 67/63 are
kept, but a reminder that single-frame detections on a busy scene are noisy.

---

## 2.3 Finding 2 — phones are missed because the input is downscaled

`ObjectDetector.detect()` never passed `imgsz`, so YOLO ran at its **640**
default. A 1280-wide source is letterboxed down to 640 — roughly a **2× linear
downscale**, which quarters the pixel area of a small, distant object. A phone
held low by a student across the room simply has too few pixels left to clear
any sane confidence. The standalone tester (`test-photos/run.py`) had used
`imgsz=1280` all along, which is why it "saw" more than the pipeline did — a
direct symptom of the missing `imgsz` in production.

Fix: make `imgsz` configurable and default it to **960** (balanced), with the
per-class laptop gate raised in lockstep so the extra resolution does not flood
the report with paper FPs. (1280 is available via `AI_OBJECT_IMGSZ` for maximum
recall; cost ~3.5–4× per frame.)

---

## 2.4 Finding 3 — the drawn box was never the detection box (the core bug)

This is the cause of the user's exact symptom — a green box on an empty wall.

The data path **dropped the detection's bounding box**:

* `ObjectDetector`/`PoseAnalyzer` produced a correct bbox per detection.
* But `AlertEvent` (the consolidated event) **had no bbox field**, so
  `consolidate_events()` discarded it. By the time an event reached `services`,
  only `(behavior_type, confidence, timestamp)` survived.
* To draw evidence, `_attach_alert_evidence()` then ran a **completely separate**
  OpenCV **Haar-cascade frontal-face detector** (`track_persons`) and drew the
  **time-nearest face box** it found — or, when it found none, a hard-coded
  **centred fallback box** (`_fallback_bbox`, 34%×46% of the frame).

On a classroom full of bookshelves, wall posters and the Washington/Lincoln
portraits, the Haar cascade fires on **furniture and patterns**, so the "flagged
person" box landed on a **bookshelf, a blank wall, or empty space** — and the
`Person N — <behaviour>` label got glued onto it. The `???` in the label was an
**em-dash** (`—`) that OpenCV's Hershey vector font cannot render.

So the picture the user saw was a *false* laptop detection (Finding 1) whose box
was drawn at a location **unrelated even to the false detection** (Finding 3).
The project's own audit had already flagged the mechanism: `critical_problems.md`
**H6** ("Face Tracker Runs Haar Cascade Independent of YOLO") and **H11**
(degenerate pose boxes merging two people).

By contrast, the **pose model is excellent**: at t = 123 s it boxed every
student cleanly at **0.62–0.94** confidence with full skeletons
(`class_pose_123s.jpg`). The accurate person geometry was available all along —
it just was not the thing being drawn.

---

## 2.5 Finding 4 — looking-away cannot be done with 2D keypoints on this camera

After rewriting the heuristic (v2), a sanity check on the classroom frames
**over-flagged badly**: **7 of 12** people at t = 123 s and 3 at t = 45 s, most
at confidence 1.0 — including a student writing and the **teacher walking**.

Dumping the raw per-person signals (sorted left-to-right) at t = 123 s revealed
why no threshold can work on this footage:

| signal | observation |
|--------|-------------|
| `off_sh` = nose-vs-shoulder offset / shoulder-width (**V2**) | Chaotic: ranged **0.11 → 18.0**; most forward-facing students were already **> 0.55**. The elevated, angled camera projects a downward/leaning head into a large *horizontal* nose offset, so V2 is **not** pitch-immune under perspective. |
| `off_ear` = nose-vs-ear-midpoint / ear-span | **−1 for everyone** — both ears were *never* confidently visible at once, so this signal could not be computed at all on this view. |
| `ear_ratio` (**V1**) | High for nearly everyone (2.2 → **838**) with **no separation** between turned and forward students — because the oblique camera hides each forward-facing student's *far* ear anyway. |
| `eye_lo` | Also non-separating (a forward student had 0.04; another had 0.77). |

**Conclusion:** from a single oblique 2D view, "head turned toward a neighbour"
and "head down over my own paper / seated at an angle" are **not separable** by
keypoint geometry. The signals are dominated by **camera viewpoint**, not head
yaw. The heuristic is sound for a roughly **frontal** proctoring camera (where a
bowed head keeps the nose centred and a real turn hides the far ear), but it must
not run on this kind of footage.

This is an honest negative result, and it is the reason looking-away ships
**disabled by default** with the proper fix (a 3D head-pose model) documented as
follow-up.

---

## 2.6 Summary of root causes

| # | Symptom the user saw | Root cause | Fixed by |
|---|----------------------|------------|----------|
| 1 | "Laptop Detected" with no laptop | YOLO reads exam paper as laptop, FPs up to 0.79, old threshold 0.40 | Per-class laptop threshold **0.85** |
| 2 | "Where's the phone?" | Input downscaled to 640; small phones lost | `imgsz` configurable, default **960** |
| 3 | Green box on an empty wall / bookshelf | Detection bbox discarded; report drew an unrelated Haar face / fallback box | Carry the real bbox end-to-end; draw **that**; Haar demoted to avatar/context/attribution |
| 3b | `Person 1 ??? Laptop Detected` | Em-dash unrenderable in OpenCV font | `—` → `-` |
| 4 | (would-be) every student "looking away" | 2D keypoint signals contaminated by oblique camera | v2 heuristic **gated OFF by default**; head-pose model as follow-up |

Continue to [03_changes_made.md](03_changes_made.md) for the exact edits.
