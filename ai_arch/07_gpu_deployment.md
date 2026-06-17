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

> To validate the entire GPU path on a **local** NVIDIA card (e.g. an RTX 3070
> dev box) before touching AWS, see **§7.8** — it was verified end-to-end there.

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

# Replace any CPU torch with a CUDA build whose index matches your torch VERSION
# (not just the driver). For torch 2.12.x that is the cu130 (CUDA 13.0) index;
# confirm the exact wheel exists for your Python/OS at download.pytorch.org/whl.
# Uninstall the CPU build first for a clean swap (no torchaudio — unused here):
pip uninstall -y torch torchvision torchaudio
pip install torch==2.12.0 torchvision==0.27.0 \
    --index-url https://download.pytorch.org/whl/cu130

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
* **Multi-GPU / autoscaling / spot** — single instance, single worker only.
* **TensorRT export** — max throughput, but the engine is GPU-arch-specific and
  adds build complexity; not worth it for a demo.

See [06_conclusion_and_followups.md](06_conclusion_and_followups.md) for the
detection/accuracy follow-ups (these are orthogonal — speed, not accuracy).

---

## 7.8 Local GPU testing on a dev box (verified on an RTX 3070)

You can validate the **entire** GPU code path on any local NVIDIA card before
touching AWS. This was verified end-to-end on a Windows 11 + **RTX 3070** (8 GB,
Ampere) box — a faithful proxy for the T4 on **correctness and VRAM**, but **not
speed** (see the caveat).

**1. Install a version-matched CUDA torch.** The §7.1 trap applies locally too: a
default install leaves `torch x.y.z+cpu`. Match the wheel index to your torch
*version*, not just the driver — for **torch 2.12.x the CUDA wheels live only on
the cu130 (CUDA 13.0) index** (cu128/cu129 stop at older torch). On Windows +
Python 3.13, into the project venv:

```
.venv\Scripts\python -m pip uninstall -y torch torchvision torchaudio
.venv\Scripts\python -m pip install torch==2.12.0 torchvision==0.27.0 --index-url https://download.pytorch.org/whl/cu130
```

(No torchaudio wheel on cu130, and the pipeline doesn't use it. A modern driver —
here 610.47, advertising CUDA 13.3 — satisfies the CUDA-13.0 runtime bundled in
the wheel, so you do **not** install a separate CUDA Toolkit. If pip can't find a
wheel for your interpreter, list real ones with
`pip index versions torch --index-url https://download.pytorch.org/whl/cu130`.)

**2. Verify, then smoke-test** with the bundled harness, which drives the *real*
detectors, the model cache, FP16 gating and device resolution — not just
`torch.cuda`:

```
.venv\Scripts\python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
.venv\Scripts\python test-photos\gpu_smoketest.py        # defaults to bus.jpg
```

Measured on the RTX 3070 (imgsz 960, FP16), both YOLO11x models resident:

```
resolved device  : 0  (GPU)  NVIDIA GeForce RTX 3070
FP16 (half)      : object=True pose=True -> applied=True
object detect    :  49.0 ms
pose analyze     :  52.2 ms          (~100 ms per sampled frame, both models)
peak VRAM        : reserved 0.57 GB  (torch allocator)
```

So the two models in FP16 use ~0.57 GB in torch's allocator; with the fixed
CUDA/cuDNN context (~0.6–1 GB, tracked by the driver, not torch) the whole process
footprint is ~1.5 GB — against ~6 GB free on the 8 GB card that fits with multi-GB
headroom, and trivially on the 16 GB T4.

**Speed caveat — the 3070 is the *faster* card.** Its FP16 tensor throughput is
~2–3× the 70 W T4's, so treat any local per-frame timing as a **floor**: budget
the production T4 at roughly 2–3× slower (≈150–300 ms per sampled frame for both
models). The *accuracy* you observe locally is representative (FP16 numerics are
comparable across Turing and Ampere); only latency differs. Size the frame-sample
rate and any job timeout against the **T4**, not the 3070.

---

## 7.9 NVENC video re-encode (GPU H.264 encoding)

The annotated full-length video and each evidence clip are re-encoded to
browser-playable H.264 ([`_reencode_h264`](../../no-more-cheaters-backend/nomorecheaters/apis/ai/face_tracker.py)).
By default (`AI_NVENC=auto`) the pipeline uses the GPU's hardware encoder
**NVENC** (`h264_nvenc`) when it is available — far faster than CPU `libx264` on
long recordings — and **always falls back to libx264** if NVENC isn't usable
(missing encoder, exhausted session, driver mismatch). NVENC can only *speed up*
a re-encode here, never break one. This matters because once inference is
GPU-fast, the full-length re-encode is the next-largest cost on long videos.

**Requirement: an ffmpeg built with `h264_nvenc`.** The pipeline auto-detects it
(reads `ffmpeg -encoders` once, caches the result); if the encoder isn't listed
it silently uses libx264. The **bundled `imageio-ffmpeg` does NOT include NVENC**,
so for GPU encoding you need a system ffmpeg with it on `PATH`:

```bash
ffmpeg -hide_banner -encoders | grep nvenc      # must list h264_nvenc
```

On the g4dn DLAMI, install/verify such an ffmpeg — a static build with NVENC
(e.g. BtbN/FFmpeg-Builds or johnvansickle.com), or the distro package if it was
built with nvenc. No capable ffmpeg → the pipeline just encodes on CPU;
correctness is unaffected, only speed.

Controls:

| Env var | Default | Meaning |
|---------|---------|---------|
| `AI_NVENC` | `auto` | `auto` → NVENC when the ffmpeg binary advertises `h264_nvenc`, else libx264. `off` → always libx264 (CPU). |

Verified on an RTX 3070 with an NVENC-capable ffmpeg: the re-encode logs
`Video re-encode codec: h264_nvenc (GPU)` and produces a valid H.264 file. On a
box whose ffmpeg lacks NVENC it logs `libx264 (CPU)` and proceeds unchanged.

