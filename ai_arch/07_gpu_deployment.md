# 7. GPU Deployment (AWS g4dn / NVIDIA T4)

How to run the AI pipeline on a GPU so analysis is fast. Target for the MVP demo:
a single **g4dn** instance (NVIDIA **T4**, 16 GB) on a **Deep Learning AMI**,
processing **one video at a time** (a single serialized worker), with **FP16**
inference on. Scalability (multi-GPU, autoscaling, batching, NVENC) is explicitly
out of scope here — see §7.6.

---

## 7.1 What the code already does (so deployment is mostly ops, not code)

The pipeline is device-agnostic and was built to light up on a GPU with almost no
code change:

* [`config.resolve_device()`](../../no-more-cheaters-backend/nomorecheaters/apis/ai/config.py)
  picks CUDA device `0` when `torch.cuda.is_available()` (honouring `AI_DEVICE`),
  caches the choice, and **logs which device it resolved to** (incl. the GPU
  name) so you can confirm it's on the T4.
* Both detectors pass `device=` to every `predict()` call and apply **FP16**
  (`half=True`) — but only when the resolved device is a GPU (FP16 on CPU raises
  in torch). FP16 now defaults **on** (`AI_HALF=true`); it is simply ignored on
  CPU.
* Loaded models are **cached per process and reused across jobs**
  ([`model_cache.py`](../../no-more-cheaters-backend/nomorecheaters/apis/ai/model_cache.py)),
  and **warmed** with a one-shot dummy inference on first load, so a long-lived
  worker doesn't reload ~230 MB of weights or pay cuDNN-autotune latency on every
  video.

The single biggest trap is that **none of this runs on the GPU unless a CUDA
build of torch is installed.** The default `pip install ultralytics` can pull a
**CPU-only** torch (`2.x.y+cpu`), in which case the code silently and correctly
falls back to CPU — fast to start, slow to run. **Verify CUDA explicitly** (§7.3).

---

## 7.2 Launch the instance

* **Instance:** `g4dn.xlarge` (1× T4, 4 vCPU, 16 GB RAM) is enough for the demo.
* **AMI:** a current **AWS Deep Learning AMI** (e.g. *Deep Learning OSS Nvidia
  Driver AMI, Ubuntu*) — it ships the NVIDIA driver + CUDA so you don't install
  drivers by hand.
* **Storage:** ≥ 30 GB (weights, media, the annotated-video re-encode scratch).

Confirm the GPU is visible:

```bash
nvidia-smi          # should list "Tesla T4" and a driver/CUDA version
```

## 7.3 Install the app with a CUDA torch

```bash
# in the project venv (or the DLAMI's conda env)
pip install -r requirements.txt

# Replace any CPU torch with the CUDA build matching the AMI's CUDA major
# (cu124 shown; use the index URL for the CUDA your driver supports):
pip install --upgrade --force-reinstall torch \
    --index-url https://download.pytorch.org/whl/cu124

# VERIFY — this must print: True  Tesla T4
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU')"
```

> Do **not** pin a CUDA wheel in `requirements.txt`; it would break CPU-only and
> Windows dev installs. Keep the CUDA install a deployment step (this doc).

## 7.4 Provision the weights (avoid a first-job download)

Ultralytics downloads `yolo11x.pt` / `yolo11x-pose.pt` (~230 MB) on first use.
For a predictable demo, place them ahead of time so the first analysis isn't
blocked on a download:

* Easiest: run the worker from `no-more-cheaters-backend/` where the weights are
  already committed — ultralytics resolves the bare filename from the CWD; **or**
* Set absolute paths: `AI_OBJECT_MODEL=/opt/nmc/yolo11x.pt`,
  `AI_POSE_MODEL=/opt/nmc/yolo11x-pose.pt`.

## 7.5 Environment & run the worker

Set in the backend environment / `.env`:

```bash
AI_DEVICE=auto        # or 0 — both resolve to the T4
AI_HALF=true          # FP16 (default); ~2x throughput, GPU-only, ignored on CPU
AI_OBJECT_IMGSZ=960   # keep the balanced default; 1280 only if phones are missed
AI_WARMUP=true        # one-shot warmup on model load (default)

RQ_ASYNC=true         # process analysis on the django-rq queue (needs Redis)
```

Run **exactly one** worker on the GPU (serialized = no VRAM contention):

```bash
redis-server &                                   # or a managed Redis
python manage.py rqworker default                # ONE process only
```

> Running multiple `rqworker`s on a single T4 would load multiple copies of both
> XL models and can OOM the 16 GB card. For the MVP, keep it to one.

## 7.6 Verify it's actually on the GPU

Trigger an analysis, then check:

* **Worker log** shows the resolved device and warmup, e.g.:
  `YOLO inference device: GPU 0 - Tesla T4 (torch 2.x.y, CUDA 12.4)` and
  `Warmed up YOLO model (device 0, imgsz 960)`.
* `nvidia-smi` during a run shows the python process holding ~2–3 GB and GPU
  utilisation spiking.
* The report/job `metadata.processing_time_seconds` drops by ~an order of
  magnitude vs the CPU baseline (two YOLO11x passes per sampled frame go from
  ~seconds to ~tens of ms each).

VRAM budget: two YOLO11x models in FP16 ≈ a couple of GB — comfortable on the
16 GB T4 for one worker.

---

## 7.7 Explicitly NOT done for the MVP (known, deferred)

These are real throughput levers, left out to keep the demo simple and reliable:

* **Batched inference** — frames are still processed one at a time. Batching N
  sampled frames per `predict()` call would raise GPU utilisation further; it
  needs code changes in both detectors + the orchestrator.
* **NVENC video re-encode** — the annotated-video re-encode still uses **CPU
  `libx264`** ([`_reencode_h264`](../../no-more-cheaters-backend/nomorecheaters/apis/ai/face_tracker.py)
  hardcodes it; the bundled imageio-ffmpeg can't do NVENC anyway). On a T4 a
  system ffmpeg built with `h264_nvenc` would speed this up, but CPU encode is
  fine for a demo. After inference is GPU-fast, this re-encode becomes the
  dominant cost on long videos.
* **Multi-GPU / autoscaling / spot** — single instance, single worker only.
* **TensorRT export** — max throughput, but the engine is GPU-arch-specific and
  adds build complexity; not worth it for a demo.

See [06_conclusion_and_followups.md](06_conclusion_and_followups.md) for the
detection/accuracy follow-ups (these are orthogonal — speed, not accuracy).
