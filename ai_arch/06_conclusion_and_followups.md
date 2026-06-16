# 6. Conclusion & Follow-ups

## 6.1 Conclusion

The reported symptom — *a green box on an empty wall labelled `Person 1 ???
Laptop Detected`, with no laptop or phone in sight* — was **not one bug but
three problems stacked together**, and the investigation confirmed each
empirically on the real footage:

1. **The detector was mis-tuned.** It ran at 640 px (downscaling away small
   phones) and used one global 0.40 confidence, which let YOLO's habit of
   reading **exam paper as "laptop"** (false positives up to **0.79**) through
   as real alerts.
2. **The overlay drew the wrong box entirely.** The real detection box was
   discarded inside the pipeline; the report instead drew an unrelated
   **Haar-cascade face box** (or a centred fallback), which on a classroom
   landed on bookshelves and blank walls. The `???` was an unrenderable em-dash.
3. **Looking-away is not solvable with 2D keypoints on this camera.** On an
   oblique/overhead view, perspective makes the head-pose signals fire on
   forward-facing students, so the heuristic flagged most of the class.

After the fix:

* Object detections carry their **real bounding box end-to-end**, and the report
  draws **that** — verified to hug the detection rather than float on a wall.
* **Class-specific thresholds** (phone 0.30, laptop 0.85) plus a configurable,
  higher inference resolution (960) eliminate the paper-as-laptop alerts while
  giving phones a real chance; on the test footage the room now correctly
  produces **zero false alerts**.
* The **looking-away heuristic was rebuilt** (more robust, fixes the
  two-person-merge bug) but is **gated OFF by default**, because shipping a
  detector that flags every diligent student would be worse than being honest
  about the camera limitation.
* The label `???` is fixed; the standalone tester matches production; **58/58**
  unit tests pass.

The honest bottom line for this particular video: it contains **no phones and no
laptops**, so a correct system reports nothing — which is now exactly what
happens. The value delivered is that when a real device *is* present, it will be
detected at a sensible threshold and **boxed in the right place**.

---

## 6.2 What is fixed vs. what is a known limitation

| Area | Status |
|------|--------|
| Phone/laptop **box placement** in evidence images | ✅ Fixed — draws the real detection box |
| Paper-as-laptop **false positives** | ✅ Fixed — laptop keep-threshold 0.85 |
| Small/distant **phone recall** | ✅ Improved — `imgsz` 960 (tunable to 1280); still bounded by camera resolution |
| `Person N ??? ...` label | ✅ Fixed — em-dash → hyphen |
| Two looking-away students merged (H11) | ✅ Fixed — real keypoint-derived person box |
| **Looking-away on an oblique camera** | ⚠️ Known limitation — gated OFF; needs a frontal camera or a 3D head-pose model |
| Per-student **attribution** of object alerts | ⚠️ Best-effort — nearest tracked face; only as good as Haar face detection in a busy room |

---

## 6.3 Prioritised follow-up work

### P1 — Reliable looking-away via a 3D head-pose model
The proper fix for any camera angle is a dedicated yaw/pitch/roll estimator
(**6DRepNet** or **L2CS-Net**, or MediaPipe FaceMesh as a lighter option) run on
each person crop from the pose boxes, flagging sustained yaw beyond a threshold
(ideally relative to the person's own baseline orientation). Keep it behind the
existing `AI_ENABLE_LOOKING_AWAY` flag. This decouples the signal from camera
viewpoint and removes the limitation in §6.2.

### P2 — Better object→person attribution
Object alerts currently attach to the spatially nearest **Haar** face, which is
unreliable in a busy room. Deriving person identity from the **pose** person
boxes (which are accurate) — and tracking those left-to-right across the video —
would give stable, correct "Person N" grouping for object alerts and remove the
third (Haar) video scan entirely.

### P3 — Maximise phone recall if needed
If real phones at distance are still missed at `imgsz=1280`, the documented next
step is **SAHI tiled inference** (≈512–640 px slices, ~0.2 overlap) to run the
detector on high-resolution tiles. This trades runtime for small-object recall.

### P4 — Test the real pipeline (audit H10)
All unit tests mock `analyze_video`; the real detection code has no automated
coverage. Add an integration test that runs the pipeline on a tiny committed clip
with a known object and asserts at least one correctly-boxed detection. This
would guard against an Ultralytics/OpenCV upgrade silently breaking detection.

---

## 6.4 Related, still-open audit items (out of scope here)

These are documented in [`../critical_problems.md`](../critical_problems.md) and
were **not** part of the detection/overlay fix:

* **C1** — the AI-threshold slider in the frontend is not wired to the pipeline
  (placebo). The pipeline reads its own env-var config; nothing bridges the two.
* **C2** — `MULTIPLE_FACES`, `OTHER_PERSON`, `OBJECT_DETECTED` behaviour types
  have **no detector** and will always count zero.
* **C5 / H1** — the report cheating-probability uses binned severity weights and
  an averaging model that can *fall* as more evidence is added.
* **C3 / C4** — a DB transaction is held open during minutes of CPU/I-O, and old
  evidence files are orphaned on re-analysis.

Addressing C1 (bridge the slider to `AI_*_CONFIDENCE`) and C2 (either implement
the detectors or remove the dead behaviour types) would be the natural next
milestones once detection accuracy itself is trusted.

---

*Dossier authored 2026-06-16 during the detection/overlay investigation; see the
other files in this directory for the full architecture, evidence, edits,
configuration, and verification.*
