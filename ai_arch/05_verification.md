# 5. Verification

The fix was verified three ways: the unit suite (no contract broken), the
empirical detector behaviour (paper-laptops gone, coordinate space confirmed),
and the full end-to-end evidence path (the green box now lands on the real
detection).

---

## 5.1 Unit suite — nothing broken

```
cd no-more-cheaters-backend/nomorecheaters
python manage.py test apis
→ Ran 58 tests in ~81s — OK
```

The suite mocks `analyze_video` with `fake_analysis_result()`, a
`SimpleNamespace` event that has **no `bbox`**. The fix relies on
`getattr(event, 'bbox', None)` and a fallback path in `_attach_alert_evidence`,
so the mock still yields exactly one Alert + Report and a `COMPLETED` session —
proving the no-detection-box path is preserved.

---

## 5.2 Coordinate-space invariant (load-bearing)

The whole overlay fix assumes YOLO boxes are in **original-frame pixels**
regardless of `imgsz`. Confirmed on the 1280×720 frame:

| imgsz | persons | max box x2 | in original 1280-wide space? |
|-------|---------|-----------|------------------------------|
| 640 | 12 | 1270 | ✅ (x2 > 640) |
| 960 | 17 | 1280 | ✅ (x2 > 960) |

A box edge exceeding the `imgsz` value proves Ultralytics rescales boxes back to
the source frame — so a carried bbox is directly drawable on the full-res frame.

---

## 5.3 Detection quality — paper-laptops gone

Running the **real** `ObjectDetector` (default config: `imgsz=960`, phone 0.30,
laptop 0.85) on the classroom frames:

```
t=45s  : OBJECTS kept = 0
t=123s : OBJECTS kept = 0
```

The six/three paper "laptops" that previously alerted are now all below the 0.85
keep-threshold and dropped. With looking-away off, the frames produce **no
alerts at all** — the correct result for a room with no phones and no laptops.

Full-clip check (8 s classroom clip, default config):

```
analyze_video(clip) → EVENTS: 0, by_type: {}, raw_detections: 0
```

---

## 5.4 End-to-end overlay — the box is now on the detection

To prove the box *plumbing* (the user's core complaint), the laptop threshold was
**temporarily forced to 0.30** so a paper detection would fire, and the **real**
`build_ai_report` evidence path was run on the clip:

```
=== build_ai_report: 5 alert(s) ===
  LAPTOPS t=6s conf=0.53 person=person_1
     detection_bbox(xywh)=[378,358,35,26]  drawn bbox=[378,358,35,26]
  LAPTOPS t=1s conf=0.77 person=person_2
     detection_bbox(xywh)=[965,434,77,104] drawn bbox=[965,434,77,104]
  ... (detection_bbox == drawn bbox for every alert)
```

The generated evidence image (`diag_out/e2e_snapshot.jpg`) shows the green box
**hugging the detected desk/paper region** and labelled `Person 1 - Laptop
Detected` (em-dash fixed) — versus the original report, where the box floated
over an empty wall. `detection_bbox == drawn bbox` for every alert confirms the
metadata the report stores is exactly what is drawn.

> Note: in production (laptop 0.85) this paper detection does **not** fire — the
> forced-low threshold here exists only to exercise the drawing path. When a
> *real* phone/laptop is present, this is the path that will box it correctly.

---

## 5.5 Looking-away — why it is gated off (the negative result)

Running the v2 heuristic with `AI_ENABLE_LOOKING_AWAY=true` on the oblique
classroom footage flagged **7 of 12** people at t=123s and **3** at t=45s
(including a writing student and the walking teacher), at confidence 1.0. The
per-person signal dump showed no threshold separates turned from forward on this
camera (see [02_investigation_findings.md](02_investigation_findings.md) §2.5).
Hence the default-off gate. On a frontal camera the same heuristic is expected to
behave; that path is opt-in and must be recalibrated per camera.

---

## 5.6 Reproduce it yourself

```bash
cd no-more-cheaters-backend

# 1) unit suite
nomorecheaters && python manage.py test apis        # 58 OK

# 2) detector on any frame / folder (standalone, matches production thresholds)
python test-photos/run.py test-photos/              # objects + facing/sideways/turned

# 3) full pipeline on a clip (no Django models needed for analyze_video)
python - <<'PY'
import os, sys
os.environ['AI_ENABLE_LOOKING_AWAY'] = 'false'
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'nomorecheaters.settings')
sys.path.insert(0, 'nomorecheaters'); import django; django.setup()
from apis.ai import analyze_video, config
print('imgsz', config.OBJECT_IMGSZ, 'thr', config.CLASS_CONFIDENCE)
r = analyze_video('<path-to-short-clip>.mp4', annotate=False)
print('events', len(r.events), r.metadata['events_by_type'])
PY
```

The diagnostic scripts used during the investigation were scratch files and were
removed after use; the proof images remain under
`no-more-cheaters-backend/diag_out/` (untracked — safe to delete).
