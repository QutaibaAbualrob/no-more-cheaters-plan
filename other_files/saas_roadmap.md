# No More Cheaters — Future Work: SaaS Roadmap

> **Audience:** Investors, product roadmap, post-graduation planning
> **Scope:** Details for areas #1 (Security & Auth), #2 (Billing), #4 (Video & AI), #6 (Infrastructure)

---

## Area 1: Security & Auth

### 1.1 OTP / Two-Factor Authentication

#### What
When a user signs in from an unrecognized device or IP, send a one-time code via SMS (Twilio) or authenticator app (TOTP). Without the code, the session is blocked.

#### Backend Changes

| Piece | What to Build | Effort |
|---|---|---|
| **TOTP model** | `UserOTP` table: `user (FK), secret (encrypted), method (SMS/TOTP/EMAIL), is_primary, created_at` | Small |
| **SMS gateway** | Twilio integration in `services.py`: `send_sms(phone, message)` — requires US/UK phone number purchase | Small |
| **TOTP library** | `pyotp` + `qrcode` for Google Authenticator-style provisioning URIs | Small |
| **Enforce middleware** | Custom DRF authentication class: if user has OTP enabled, a `POST /api/auth/otp/verify/` must be called within 60s before the session token becomes valid | Medium |
| **OTP endpoints** | `POST /api/auth/otp/enable/` (returns provisioning URI), `POST /api/auth/otp/verify/` (submits code), `DELETE /api/auth/otp/disable/` | Small |
| **Trusted devices** | `TrustedDevice` model: `user, device_fingerprint, expires_at`. Skip OTP for 30 days on trusted devices | Small |
| **Backup codes** | Generate 8 single-use backup codes on OTP enable, store hashed | Small |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **OTP setup page** in Profile: QR code + manual key + test field | Small |
| **OTP challenge modal** on login: code input, resend countdown | Small |
| **Trust this device** checkbox | Tiny |
| **Backup codes display** — show once, download TXT | Small |

#### Cost
- Twilio SMS: ~$0.0079 per SMS (US) / ~$0.05 (international)
- Free for TOTP-only (Google Authenticator, Authy)
- Phone number purchase: ~$1-2 one-time

#### Priority
🟠 Should build before paid tiers go live.

---

### 1.2 Social Login (Google / Microsoft)

#### What
"Sign in with Google" and "Sign in with Microsoft" buttons on the login page. Links existing accounts by email, or creates new ones.

#### Backend Changes

| Piece | What to Build | Effort |
|---|---|---|
| **allauth socialaccount** | Enable `allauth.socialaccount` and add Google + Microsoft providers in settings.py | Small |
| **Google OAuth client** | GCP console → OAuth 2.0 credentials → client ID + secret → env vars | Small |
| **Microsoft Entra ID** | Azure portal → App registration → tenant ID + client secret → env vars | Small |
| **Social serializer** | Extend `CustomRegisterSerializer` to handle role assignment for social-first signups (default INSTRUCTOR, allow upgrade in profile) | Small |
| **Social login URL pattern** | `GET /api/auth/social/google/login/`, callback, etc. | Tiny |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **Google sign-in button** with Google icon on SignIn/SignUp | Small |
| **Microsoft sign-in button** with Microsoft icon | Small |
| **Account linking UI** — if email already exists, prompt to link instead of duplicate | Medium |
| **Social disconnect** in Profile: "Unlink Google" button | Small |

#### Priority
🟠 Nice-to-have — reduces sign-up friction significantly.

---

### 1.3 Session Management

#### What
Users can view all active sessions (browser, device, IP, last-active time) and revoke individual sessions.

#### Backend Changes

| Piece | Effort |
|---|---|
| **Session tracking** — every login creates a `UserSession` row (device fingerprint, IP, user agent, token prefix, expires_at) | Medium |
| **`GET /api/auth/sessions/`** — list my active sessions | Small |
| **`DELETE /api/auth/sessions/<id>/`** — revoke a session (delete token) | Small |
| **`DELETE /api/auth/sessions/`** — revoke all other sessions (rotate token) | Small |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **Session list** in Profile: table with device, browser, IP, last active, "Revoke" button | Small |
| **"Revoke all other sessions" button** — prompts confirmation | Small |

#### Priority
🟢 Important for security-conscious institutions (AAUP).

---

### 1.4 Rate Limiting & Brute-Force Protection

#### What
Limit login attempts per email per minute. Block IPs after 10 consecutive failures.

#### Backend Changes

| Piece | Effort |
|---|---|
| **django-ratelimit** middleware on auth endpoints | Tiny |
| **Login attempt tracking** — `LoginAttempt` model: email, IP, timestamp, success | Small |
| **Auto-block IP** after N failures within a rolling 15-min window | Small |
| **Admin alert** — email sent to admin when an IP is blocked | Small |

#### Priority
🔴 Must-have before any public deployment. Already partially relevant.

---

## Area 2: Billing (Stripe)

### 2.1 Subscription Tiers

#### Proposed Tiers

| Tier | Price | Videos/Month | Users | Analysis Speed | Storage |
|---|---|---|---|---|---|
| **Free** | $0 | 3 | 1 instructor | CPU (slow) | 500 MB |
| **Starter** | $29/mo | 50 | 5 instructors, 1 dean | GPU priority | 10 GB |
| **Professional** | $99/mo | Unlimited | 20 instructors, 2 deans | GPU priority + parallel | 50 GB |
| **Enterprise** | Custom | Unlimited | Unlimited | Dedicated GPU instance | Custom |

#### Backend Changes

| Piece | What to Build | Effort |
|---|---|---|
| **Stripe webhook** | `POST /api/billing/webhook/` — handles `customer.subscription.created/updated/deleted`, `invoice.paid`, etc. | Medium |
| **Subscription model** | `Subscription` table: `institution (FK), stripe_subscription_id, status, tier, current_period_start/end` | Small |
| **Usage tracking** | `UsageRecord` table: `subscription, month, year, video_count, analysis_count, storage_bytes` | Small |
| **Tier enforcement middleware** | Before video upload: check `usage.video_count < tier.limit`. Before analysis: check tier. Return 402 Payment Required if exceeded | Medium |
| **Stripe product sync** | Create products + prices in Stripe dashboard → mirror in DB as `BillingPlan` | Small |
| **Invoice history** | `Invoice` model synced from Stripe webhooks: `stripe_invoice_id, amount, status, paid_at, pdf_url` | Small |
| **Promo code support** | `PromoCode` table: `code, discount_percent, max_uses, expires_at` | Small |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **Pricing page** (`/pricing`) — 3 tiers + "Contact us" for enterprise | Medium |
| **Checkout flow** — Stripe Checkout or Stripe Elements (credit card form) | Medium |
| **Billing settings** in `/app/billing` — current plan, usage meter, invoice list, cancel/upgrade/downgrade | Medium |
| **Upgrade prompt** — when hitting tier limits, show upgrade modal | Small |
| **Payment method management** — Stripe Customer Portal (redirect) | Small |

#### Backend Architecture

```
User clicks "Subscribe" → POST /api/billing/create-checkout-session/
                           → Django creates Stripe Checkout Session
                           → Returns session.url
                           → Frontend redirects to Stripe

Stripe calls back → POST /api/billing/webhook/
                    → Verify signature
                    → Update Subscription status
                    → Activate/Deactivate features
                    → Send confirmation email
```

#### Cost
- Stripe: 2.9% + $0.30 per transaction
- No monthly fee

#### Priority
🔴 Essential for monetization. Build after MVP stabilizes.

---

### 2.2 Usage-Based Add-Ons

| Add-On | Price | What It Unlocks |
|---|---|---|
| **Extra video quota** | $10 per 10 videos | Beyond tier limit |
| **Additional instructor seat** | $5/user/mo | Extra users beyond tier |
| **AI model customization** | $200 one-time | Fine-tune on institution's exam footage |
| **White-label branding** | $50/mo | Remove "No More Cheaters" branding |
| **API access** | $20/mo | Rate-limited API keys for programmatic access |

---

## Area 4: Video & AI

### 4.1 Batch Upload & Processing

#### What
Upload 10+ videos at once, analyze all in parallel, see aggregated results.

#### Backend Changes

| Piece | Effort |
|---|---|
| **Batch upload endpoint** — `POST /api/videos/batch-upload/` accepts multiple files, creates sessions + AnalysisJobs in a single transaction | Medium |
| **Parallel analysis** — each upload fans out to a separate rq job; async queue handles concurrency | Small (already architecture supports) |
| **Batch status endpoint** — `GET /api/videos/batch-status/<batch_id>/` returns aggregate progress (X of Y done) | Small |
| **Batch model** — `BatchUpload` table: `instructor, status, total, completed, failed, created_at` | Small |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **Multi-file upload zone** — drag-and-drop for 10+ files, file list with checkboxes | Medium |
| **Batch progress dashboard** — progress bar per file, status grid, ETA | Medium |
| **Batch results view** — aggregated probability distribution chart, per-video drill-down | Medium |

#### Priority
🟠 High-value for instructors with large classes.

---

### 4.2 Live Proctoring (Real-Time)

#### What
Instead of recording then uploading later, the system watches the student's webcam live during the exam and flags suspicious behavior in real time.

#### Backend Changes

| Piece | Effort |
|---|---|
| **WebSocket endpoint** — Django Channels + Redis pub/sub for frame streaming | Large |
| **Frame receiver** — accepts JPEG frames via WebSocket, runs YOLO inference, returns alerts in <500ms | Large |
| **Live session model** — `LiveSession` table: `exam, student, status (ACTIVE/PAUSED/ENDED), started_at` | Medium |
| **Alert accumulator** — real-time alert stream → persisted to Alert table on session end | Medium |
| **WebRTC signaling server** — optional, if using browser capture → server relay | Large |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **Live proctoring dashboard** — grid of student webcam thumbnails, red badge on flagged students | Large |
| **Per-student detail** — live alert feed, snapshot history, probability meter | Large |
| **Instructor controls** — pause/end session, send warning to student, mark student as reviewed live | Medium |

#### Architecture

```
Student Browser                  Server                    Instructor Dashboard
───────          ──────                    ───────
Webcam frame ──► WebSocket ──► YOLO inference        ──► Alert stream
every 2s                  │                        │
                          ▼                        ▼
                    Frame discarded          Alert pushed to instructor
                    (not stored)             via WebSocket
```

#### Cost
- GPU must stay hot 24/7 during exams (higher EC2 cost)
- Bandwidth: ~50-100 MB per student per hour
- Requires 2x-3x GPU capacity vs recorded analysis

#### Priority
🟢 Differentiator — what separates a "real" proctoring system from a review tool. Post-MVP.

---

### 4.3 Cloud Video Storage (S3/CDN)

#### Current
Videos stored on EC2 local disk, served through Nginx. Single point of failure, no CDN, limited space.

#### Target

| Piece | Effort |
|---|---|
| **Django storages** → `boto3` S3 backend. Files saved to S3 on upload, served via CloudFront CDN | Medium |
| **Presigned URLs** — `Video.file_url` returns time-limited S3 presigned URL instead of local path | Small |
| **Lifecycle rules** — S3 bucket automatically moves files to Glacier after 30 days, deletes after 90 | Small |
| **Multi-region** — optional S3 Cross-Region Replication for disaster recovery | Medium |

#### Cost
- S3 Standard: ~$0.023/GB/mo
- CloudFront: ~$0.085/GB egress
- S3 Glacier Deep Archive: ~$0.001/GB/mo (for compliance retention)

#### Priority
🟠 Infrastructure upgrade needed before scaling past 100 videos.

---

### 4.4 Report Export (PDF / CSV)

#### What
Download analysis reports as PDF (human-readable) or CSV (data for spreadsheets).

#### Backend Changes

| Piece | Effort |
|---|---|
| **PDF generation** — `weasyprint` or `reportlab`: renders session report with probability gauge, alert table, summary | Medium |
| **CSV export** — `GET /api/history/export/csv/?exam_id=X` returns CSV of all alerts | Small |
| **PDF endpoint** — `GET /api/reports/<id>/pdf/` returns PDF download | Small |
| **Batch export** — `POST /api/history/export/` accepts date range + exam filter | Medium |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **Download buttons** on AnalysisReports page and History page | Small |
| **Export modal** — select format (PDF/CSV), date range, exams | Small |

#### Priority
🟢 Important for professor review — they want physical evidence.

---

### 4.5 Custom AI Training Per Institution

#### What
Each institution can upload 50-100 labeled exam frames, fine-tune YOLO to recognize their specific exam environment (desks, room layout, typical student posture).

#### Backend Changes

| Piece | Effort |
|---|---|
| **Fine-tuning pipeline** — `apis/ai/trainer.py`: loads base YOLO, trains on uploaded labeled dataset, exports custom weights | Large |
| **Training job model** — `TrainingJob` table: `institution, status, base_model, epochs, accuracy, weights_path` | Medium |
| **Training queue** — dedicated rq queue (`training`), GPU locked during training | Medium |
| **Versioned model registry** — `AIModel` table: `version, institution, weights_path, active` | Small |
| **Labeling endpoint** — `POST /api/ai/labels/` — save bounding box labels per frame | Medium |

#### Cost
- GPU instance locked for 1-3 hours per training run
- ~$2-5 per training run on g4dn.xlarge

#### Priority
🟢 Enterprise tier only. Heavy effort, high value for large institutions.

---

## Area 6: Infrastructure

### 6.1 Docker / Kubernetes

#### Current
Manual setup on EC2: install Python, clone repo, run Gunicorn. No containerization.

#### Target

| Piece | Effort |
|---|---|
| **Dockerfile** — multi-stage build: Python base → install deps → copy app → Gunicorn entrypoint | Small |
| **docker-compose.yml** — Django + RQ worker + Redis + Nginx (for local development) | Small |
| **eks/Dockerfile** — GPU-enabled Dockerfile with CUDA + ultralytics | Small |
| **Helm chart** — for Kubernetes deployment: deployments for web + worker + Redis, HPA for auto-scaling | Large |
| **GitHub Actions CI** — build Docker image → push to ECR → deploy to EKS | Medium |
| **.dockerignore** + multi-arch builds for arm64 (Apple Silicon devs) | Small |

#### Benefits
- Reproducible builds across dev/staging/prod
- 5-minute deploy instead of 1-hour manual setup
- Auto-scaling: more rq workers during exam season, scale to zero off-season

#### Priority
🟠 Before going multi-tenant with more than 10 institutions.

---

### 6.2 Auto-Scaling

#### Architecture

```
During exam season (peak):
  ┌──────────────────────────────┐
  │  ALB (Application LB)        │
  ├────────┬────────┬────────────┤
  │ Web    │ Web    │ Web        │  ← scale out to 3-6
  │ Node 1 │ Node 2 │ Node 3     │     web containers
  ├────────┴────────┴────────────┤
  │  RQ Worker Pool              │
  ├──────┬──────┬──────┬─────────┤
  │ GPU1 │ GPU2 │ CPU3 │  CPU4   │  ← dedicated GPU workers
  └──────┴──────┴──────┴─────────┘       for analysis
```

| Piece | Effort |
|---|---|
| **ECS/EKS auto-scaling** — CPU/memory-based scale-out for web tier, queue-length-based for workers | Medium |
| **GPU spot instances** — 60-70% cheaper than on-demand, with graceful shutdown for workers | Medium |
| **Database connection pooling** — PgBouncer or RDS Proxy to handle 200+ concurrent connections | Medium |
| **Read replicas** — Analytics queries (dashboard, metrics) hit replica, not primary | Medium |

#### Cost Optimization
| Strategy | Savings |
|---|---|
| Spot GPU instances for batch analysis | ~60% |
| Scale workers to 0 at night | ~40% |
| RDS serverless (Aurora) | Pay per request |
| S3 infrequent access for older videos | ~30% |

#### Priority
🟢 After reaching 50+ paying institutions.

---

### 6.3 Monitoring & Alerting

| Tool | What It Monitors | Cost |
|---|---|---|
| **Sentry** | Python + JavaScript error tracking. Catches 500 errors, front-end crashes, slow API calls | Free tier (5k events/mo) |
| **Datadog** | Server metrics (CPU, RAM, disk), request latency, queue depth, database slow queries | Free up to 5 hosts |
| **Uptime Robot** | HTTPS endpoint health check — alerts when `nomorecheater.online` is down | Free (5 min intervals) |
| **Custom health endpoint** | `GET /api/health/` — returns DB connectivity, Redis ping, disk space, model loaded | Small effort |
| **Dashboard** | Grafana dashboard: active users, analysis queue depth, error rate, API latency | Medium effort |

#### Backend Changes

| Piece | Effort |
|---|---|
| **Sentry SDK** — `sentry_sdk` in Django + `@sentry/tracing` for auto-instrumentation | Small |
| **Health endpoint** — `GET /api/health/` checks DB, Redis, disk, AI model loaded | Small |
| **Structured logging** — `structlog` or `python-json-logger`: JSON-formatted logs for Datadog/CloudWatch | Small |
| **RQ monitoring** — `django-rq-scheduler` admin: view queued/failed jobs, retry failed | Small |

#### Priority
🟠 Before going live with paying customers. Without it you're blind.

---

### 6.4 API Keys for External Integrations

#### What
Universities want to integrate No More Cheaters with their own systems. Provide API keys with scoped permissions.

#### Backend Changes

| Piece | Effort |
|---|---|
| **ApiKey model** — `ApiKey` table: `institution, key_hash (SHA-256), prefix (first 8 chars), permissions (JSON), expires_at, created_at` | Small |
| **API key auth class** — custom DRF authentication that validates `Authorization: Bearer nmc_<prefix>_<secret>` | Small |
| **Key management endpoints** — `POST /api/keys/` (create), `GET /api/keys/` (list), `DELETE /api/keys/<id>/` (revoke) | Small |
| **Rate limiting per key** — `django-ratelimit` scoped to API key identity | Small |
| **Usage logging per key** — `ApiKeyLog` table: `key, endpoint, ip, timestamp, response_code` | Small |

#### Frontend Changes

| Piece | Effort |
|---|---|
| **API keys page** in Admin panel — list keys, create new, copy one-time secret, revoke | Medium |

#### Priority
🟢 Enterprise tier. Needed when institutions want to automate video upload from their LMS.

---

### 6.5 Backup & Disaster Recovery

| Piece | Effort | Frequency |
|---|---|---|
| **Automated RDS snapshots** — enable in AWS Console | Tiny | Daily |
| **S3 bucket versioning** — recover deleted/replaced video files | Small | Continuous |
| **Database backup to S3** — `pg_dump` → S3 via cron (cross-region) | Small | Daily |
| **Backup retention policy** — keep daily for 7 days, weekly for 1 month, monthly for 1 year | Small | — |
| **Disaster recovery playbook** — 1-page doc: `terraform apply` in secondary region + restore from S3 | Medium | — |
| **Recovery test** — quarterly drill: spin up staging from backup, verify data integrity | Medium | Quarterly |

#### Priority
🟢 Before any production SLA is signed.

---

## Summary: Implementation Order

### Phase 1 (MVP+ — 2 weeks)
| Priority | Item | Dependency |
|---|---|---|
| 🔴 | Rate limiting + brute-force protection | — |
| 🟠 | Docker + docker-compose | — |
| 🟠 | Sentry monitoring | — |
| 🟠 | S3 video storage | — |
| 🟠 | Stripe free + starter tiers | — |
| 🟠 | PDF/CSV report export | — |

### Phase 2 (Growth — 1 month)
| Priority | Item | Dependency |
|---|---|---|
| 🟠 | OTP/2FA | Phase 1 |
| 🟠 | Session management | Phase 1 |
| 🟠 | Batch upload + processing | Phase 1 |
| 🟠 | API keys | Phase 1 |
| 🟠 | Batch export | Phase 1 |

### Phase 3 (Scale — 2 months)
| Priority | Item | Dependency |
|---|---|---|
| 🟢 | Social login | Phase 2 |
| 🟢 | Auto-scaling (ECS/K8s) | Phase 1 Docker |
| 🟢 | Live proctoring (WebSocket) | Phase 1 |
| 🟢 | Custom AI training | Phase 1 |
| 🟢 | Helm chart | Phase 1 Docker |
| 🟢 | Disaster recovery | Phase 1 |

### Phase 4 (Enterprise — 3+ months)
| Priority | Item | Dependency |
|---|---|---|
| 🟢 | White-label branding | Phase 2 |
| 🟢 | Enterprise SSO (SAML/OIDC) | Phase 2 |
| 🟢 | Dedicated GPU per tenant | Phase 3 |
| 🟢 | On-premise deployment option | Phase 3 |
| 🟢 | SLA-backed support | Phase 3 |

---

## Cost Projection (Monthly)

| Item | Free | Starter (10 institutions) | Growth (50 institutions) | Enterprise (200+ institutions) |
|---|---|---|---|---|
| EC2 GPU | $0 | $300 | $600 | $3,000 (multi-GPU) |
| RDS MySQL | $0 | $15 | $50 | $200 (Aurora) |
| S3 + CloudFront | $0 | $10 | $50 | $500 |
| Redis | $0 | $15 | $30 | $100 (ElastiCache) |
| Stripe fees | $0 | $29 × 10 × 3.3% ≈ $96 | $99 × 50 × 2.7% ≈ $1,337 | Custom |
| Twilio (SMS) | $0 | $20 | $100 | $500 |
| Monitoring | $0 | $0 (free tier) | $100 | $500 |
| **Total infra** | **$0** | **~$456** | **~$2,267** | **~$7,800** |
| **Revenue** | $0 | **$2,900** | **$49,500** | **$200,000+** |
| **Gross margin** | — | **~84%** | **~95%** | **~96%** |
