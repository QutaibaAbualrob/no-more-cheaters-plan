# No More Cheaters — AI Architecture Dossier

This directory documents the **AI video-analysis subsystem** of No More Cheaters
in depth: how it is built, what was found to be wrong with its detection and
evidence overlays, exactly what was changed to fix it, how the fix was verified,
and what remains as known limitations and future work.

It was produced during the 2026-06-16 investigation that began from a simple
user observation: report evidence images showed a **green box floating over an
empty wall** labelled `Person 1 ??? Laptop Detected`, when no laptop (and no
phone) was anywhere in the frame.

The short version: there were **two independent bugs stacked on top of each
other**, plus a **camera-geometry limitation**:

1. **Detection quality** — the object detector ran at too low a resolution to
   see phones, and a single global confidence threshold let YOLO's habit of
   reading bright sheets of **exam paper as "laptop"** through as real alerts.
2. **Overlay plumbing** — the *real* detection box was discarded inside the
   pipeline, and the report instead drew an unrelated **Haar-cascade face box**
   (or a centred fallback box), which landed on bookshelves and blank walls.
3. **Looking-away on an oblique camera** — 2D-keypoint head-pose signals are
   contaminated by perspective on a ceiling/angled camera, so they cannot
   distinguish "turned toward a neighbour" from "looking down at my own paper".

All three are addressed; #3 is addressed honestly by gating looking-away **off
by default** with a documented path to a proper fix.

---

## Read in this order

| # | Document | What it covers |
|---|----------|----------------|
| 1 | [01_pipeline_architecture.md](01_pipeline_architecture.md) | How the whole AI pipeline works, end to end — every module, dataclass, the data-flow, the temporal consolidation, the evidence/overlay path, and the contracts with the database, API, and frontend. |
| 2 | [02_investigation_findings.md](02_investigation_findings.md) | The empirical investigation. What YOLO *actually* detects in the real classroom video, the measured confidence distributions, and the three root causes with evidence. |
| 3 | [03_changes_made.md](03_changes_made.md) | The fix, file by file — before/after, exact behaviour change, and the rationale (including the design decisions and the adversarial-review issues that were folded in). |
| 4 | [04_configuration_reference.md](04_configuration_reference.md) | Every environment variable / tunable, its default, meaning, and recommended profiles for frontal vs oblique cameras. |
| 5 | [05_verification.md](05_verification.md) | How the fix was verified — the unit suite, the empirical before/after, and exact reproduction steps. |
| 6 | [06_conclusion_and_followups.md](06_conclusion_and_followups.md) | Conclusion, what is fixed vs. what remains a known limitation, and the prioritised follow-up work. |

---

## Scope

* **In scope / fixed here:** object-detection accuracy (phones, laptops), the
  evidence box overlay, the looking-away heuristic and its default gating, and
  the end-to-end bbox plumbing from detector to report.
* **Out of scope (documented elsewhere):** the broader code audit —
  threshold-page placebo (C1), unreachable behaviour types `MULTIPLE_FACES` /
  `OTHER_PERSON` / `OBJECT_DETECTED` (C2), the report-probability arithmetic
  (C5/H1), the transaction-held-open-during-I/O issue (C3), and orphaned media
  files (C4) — see [`../critical_problems.md`](../critical_problems.md). Those
  are real but separate from the detection/overlay problem this dossier targets.

## Code touched

All changes live in the backend repo, `no-more-cheaters-backend/`:

```
nomorecheaters/apis/ai/config.py          # tunables, class thresholds, looking-away gate
nomorecheaters/apis/ai/yolo_detector.py   # imgsz + per-class confidence filtering
nomorecheaters/apis/ai/detector.py        # carry bbox end-to-end; gate the pose pass
nomorecheaters/apis/ai/pose_analyzer.py   # v2 looking-away heuristic + real person box
nomorecheaters/apis/ai/face_tracker.py    # em-dash -> hyphen in drawn labels
nomorecheaters/apis/services.py           # draw the REAL detection box; spatial attribution
test-photos/run.py                        # standalone tester aligned to production
```

Verification status at time of writing: **58/58 backend tests pass**; the
classroom clip produces **0 false alerts** at the default thresholds.
