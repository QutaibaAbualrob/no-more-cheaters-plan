# No More Cheaters — Business & Scaling Limitations

> **Audience:** Investors, institution decision-makers, business planning
> **Perspective:** What stops this from being a sellable, scalable SaaS product *today*
> **Not covered:** Technical architecture, code quality, AI accuracy — see `limitations.md` for that

---

## 1. Cost Structure — Unit Economics Are Unknown

### 1.1 Per-Customer Cost Cannot Be Calculated

| Missing Piece | Why It Matters |
|---|---|
| No per-customer usage tracking | Can't answer "how much does institution X cost us to serve" |
| No metered billing infrastructure | Can't charge by video, student, or analysis — only flat-rate guessing |
| No storage cost attribution | Videos on shared EBS disk — don't know if one customer has 2 GB or 20 GB |
| No GPU time tracking | Don't know which customer consumed the most analysis minutes |
| No idle-cost calculation | GPU instance costs $0.526/hr whether it's analyzing 1 video or 100 |

**Business impact:** You cannot set a price with confidence. Either you overprice (lose customers) or underprice (lose money per customer).

### 1.2 Fixed Costs Are High, Variable Costs Are Opaque

| Cost Item | Monthly | Fixed or Variable |
|---|---|---|
| EC2 g4dn.xlarge (on-demand) | ~$380 | Fixed — runs 24/7 |
| RDS db.t3.micro | ~$15 | Fixed |
| Elastic IP | ~$3.6 | Fixed |
| Domain (Namecheap) | ~$1.25 | Fixed — annual |
| Resend (free tier) | $0 | Variable — covers first 3,000 emails |
| **Total baseline** | **~$400/mo** | **Bare minimum to exist** |

**Problem:** At 10 institutions × $29/mo = $290 revenue → **losing $110/mo** before any developer salary, support, or overhead.

**Break-even at $29/mo = 14 institutions.** At $99/mo = 5 institutions. But there's no sales pipeline to get either number.

### 1.3 GPU Cost Escalates Linearly With Customers

| Scenario | GPU Setup | Monthly GPU Cost | Notes |
|---|---|---|---|
| Demo (1-3 customers) | 1 g4dn.xlarge | ~$380 | Sequential analysis — queue builds up fast |
| 10 customers | 1 g4dn.xlarge | ~$380 | 10 min analysis × 10 videos/day = 1.7 hrs GPU/day ✅ |
| 50 customers | 2-3 g4dn.xlarge | ~$760-1,140 | 50 min/day of analysis per GPU. Need parallel workers |
| 200 customers | 5-7 g4dn.xlarge | ~$1,900-2,660 | Peak hours need 5+ simultaneous analyses |
| 1,000 customers | GPU-as-a-service (e.g. RunPod, JarvisLabs) | ~$0.30/hr per job | Spot instances, scale to zero when idle |

**Business impact:** At scale, the GPU becomes the single biggest cost driver and the hardest to predict. A single 10-minute spike (all instructors uploading after a final exam) requires 10× the normal capacity that sits idle the rest of the day.

---

## 2. Integration Gaps — Institutions Cannot Buy a Black Box

### 2.1 No LMS Integration Kills Enterprise Sales

| Missing Integration | What Institutions Expect | What We Offer |
|---|---|---|
| Moodle | Course roster auto-sync, grade push, assignment linking | Manual CSV upload |
| Canvas | LTI 1.3 deep linking, roster import, grade passback | Nothing |
| Blackboard | LTI launch, SIS integration, anonymous grading | Nothing |
| Google Classroom | Class import, Stream notifications | Nothing |
| Banner / PeopleSoft | Full SIS integration for student data | Nothing |

**Business impact:** A university IT department will reject any proctoring tool that doesn't integrate with their existing LMS. *Every* RFP (request for proposal) in higher ed requires LMS integration. Without LTI, you can't even get on the shortlist.

### 2.2 No SSO / Identity Federation

| Missing Feature | Why It Blocks Sales |
|---|---|
| SAML 2.0 | Every university has a single sign-on portal. Requiring a separate username+password is a dealbreaker |
| OIDC | Microsoft Entra ID / Azure AD is standard in 80% of universities. Without it, IT won't approve |
| LDAP | On-premise institutions need directory sync |
| Google Workspace sync | Many institutions use Google for Education — no "Sign in with Google" means password reset tickets from day 1 |

**Business impact:** SSO is not a "nice to have" in higher ed — it is a *procurement requirement*. Without it, the security team will block the purchase.

### 2.3 No LTI Compliance

| Requirement | Current | Needed |
|---|---|---|
| LTI 1.3 Advantage | ❌ | Standard for all LMS integrations |
| Names and Role Provisioning (NRP) | ❌ | Auto-sync course rosters |
| Assignment and Grade Services (AGS) | ❌ | Push analysis results back to gradebook |
| Deep Linking | ❌ | Embed proctoring session in LMS course page |

**Business impact:** LTI 1.3 certification costs ~$5,000/year (IMS Global membership + certification fees). But without it, the product is classified as a "third-party tool that requires manual setup" — which most universities no longer allow under their data governance policies.

### 2.4 No Calendar / Scheduling Integration

| Missing | Impact |
|---|---|
| No iCal/CalDAV sync | Deans have to manually schedule exams in two systems |
| No Outlook Calendar integration | Exam times don't appear on institutional calendars |
| No Google Calendar API | No "Add to Calendar" button on scheduled recordings |
| No room booking system integration | Can't sync hall availability with university's existing room scheduler (e.g. EMS, Ad Astra) |

---

## 3. Market & Competitive Limitations

### 3.1 Competing Against Established Platforms

| Competitor | Price | Key Advantage Over Us |
|---|---|---|
| **ProctorU** | $15-25/student/exam | Live human proctors + AI. Multi-billion dollar scale. LTI integration. 100+ university contracts |
| **Honorlock** | ~$15/student/exam | Browser lockdown + live proctoring + AI. Chromebook support. 300+ institution customers |
| **Respondus** | ~$5/student/year | LockDown Browser with Monitor. Installed on 2,400+ campuses. Deep LMS integration |
| **ExamSoft** | Custom per-institution | Full exam creation + proctoring + analytics. Used by 1,200+ programs |
| **Moodle Quiz** | Free (open source) | Built into the LMS itself. Zero adoption friction |

**Our position:** No current differentiator that would make an institution switch. "Recorded video analysis with YOLO" is a subset of what every competitor already offers.

### 3.2 No Differentiation Strategy

| Potential Differentiator | Status |
|---|---|
| Open-source core | ❌ Proprietary code |
| Significantly cheaper | ❌ No pricing defined |
| Significantly more accurate | ❌ 3 of 6 behaviors detected vs competitors' 8-12 detection types |
| Significantly easier to deploy | ❌ Requires AWS setup, GPU, domain config |
| Privacy-first (on-premise) | ❌ Only cloud option |
| Middle East / Arabic market focus | ❌ English-only UI, no Arabic support, no regional pricing |

**Business impact:** There is no clear "why us" answer for an institution evaluating proctoring tools. "It was built by a university student" is a liability in procurement, not a selling point.

### 3.3 Market Window Is Closing

| Trend | Impact on Us |
|---|---|
| Post-COVID, many universities returned to in-person exams | The huge 2020-2022 demand spike for remote proctoring has passed |
| AI detection skepticism | Students have successfully challenged AI proctoring flags in academic courts. Institutions are wary |
| Privacy backlash | Several US states (Ohio, Maryland, others) have restricted AI proctoring over biometric privacy concerns |
| Browser lockdown becoming standard | Universities prefer solutions that prevent cheating (lockdown browsers) over solutions that detect it after the fact |

---

## 4. Commercial Limitations

### 4.1 No Way to Collect Money

| Missing | Impact |
|---|---|
| No Stripe integration | Can't accept credit cards |
| No invoicing system | Can't send PDF invoices for institutional purchase orders |
| No multi-currency support | Can't charge in USD, EUR, ILS, or other currencies |
| No VAT/GST handling | Can't comply with EU VAT or other tax regimes |
| No payment terms | No net-30, net-60, or other institutional billing workflows |

**Business impact:** Even if someone wants to buy, there's no way to pay. The only option is free — which means zero revenue.

### 4.2 No Contract or Legal Framework

| Missing | Impact |
|---|---|
| No Terms of Service | No legal protection if a student sues over a false positive |
| No Privacy Policy | No documented data handling — violates GDPR, CCPA, and university data governance |
| No Data Processing Agreement (DPA) | GDPR requirement for any EU institution |
| No Service Level Agreement (SLA) | No uptime guarantee, no support response times |
| No Business Continuity Plan | If the EC2 instance goes down, there's no documented recovery process |
| No FERPA compliance documentation | US Federal law for student educational records. Required by any US institution |
| No ISO 27001 / SOC 2 | Enterprise procurement often requires these certifications (cost: $50,000-100,000 to obtain) |

**Business impact:** Without legal and compliance documentation, institutional procurement is impossible. Universities cannot sign a contract with a tool that has no ToS, no DPA, and no SLA.

### 4.3 No Support Infrastructure

| Missing | What Institutions Expect |
|---|---|
| No ticketing system | Email-only support with no SLA |
| No phone support | Critical during exams — if the system goes down mid-exam, they need immediate help |
| No documentation portal | No knowledge base, FAQ, or video tutorials |
| No status page | No `status.nomorecheater.site` to check if the system is down |
| No dedicated support team | Founder-only support — not scalable |
| No on-call rotation | System can go down Friday night and stay down until Monday |

**Business impact:** An institution with 5,000 students taking exams cannot use a tool where a Friday-night outage goes unnoticed until Monday.

### 4.4 No Sales or Onboarding Process

| Missing | Impact |
|---|---|
| No trial conversion funnel | No guided free → paid journey |
| No demo environment | Every demo requires setting up a live EC2 instance |
| No onboarding wizard | New users get dropped into an empty dashboard with no guidance |
| No training materials | No video tutorials, no quick-start guide, no "what to do if you're flagged" instructions for students |
| No institutional onboarding | No dedicated account setup call, no data migration, no LMS configuration help |

---

## 5. Geographic & Regulatory Limitations

### 5.1 Data Residency Blocks Global Scale

| Region | Problem |
|---|---|
| **European Union (GDPR)** | Videos stored on US-based AWS servers. No EU data center option. No DPA available. No right-to-deletion endpoint |
| **Israel** | Privacy Protection Authority requirements unclear. No Hebrew UI |
| **Saudi Arabia** | PDPL (Personal Data Protection Law) requires data localization. No KSA-based servers |
| **UAE** | Similar data localization requirements. No Dubai/AWS-ME region deployment |
| **Turkey** | KVKK law requires data residency. No Turkish servers |
| **California (CCPA)** | Additional consumer privacy protections for California-based students. No compliance tooling |

**Business impact:** The current single-region (US-east) deployment blocks the entire EU market, the Middle East institutional market, and any California public university.

### 5.2 Accessibility Regulations

| Regulation | Requirement | Compliance Status |
|---|---|---|
| **WCAG 2.1 AA** | Required for US federal funding (Section 508). Required in EU (EN 301 549). Required in UK | ❌ Not audited |
| **PDF/UA** | Reports must be accessible screen-reader friendly | ❌ No report export at all |
| **VPAT** | Voluntary Product Accessibility Template — required by most US state universities in procurement | ❌ Not created |

**Business impact:** Public universities receiving federal funding (which is most of them) cannot purchase non-compliant tools.

### 5.3 Biometric Privacy Laws

| State/Law | Restriction | Impact |
|---|---|---|
| **Illinois BIPA** | Requires explicit consent before collecting biometric data (face scans). Private right of action ($1,000-5,000 per violation) |
| **Texas** | Similar biometric privacy law |
| **Washington** | Similar biometric privacy law |
| **EU GDPR** | Biometric data is "special category" — requires explicit consent + legitimate interest assessment |

**Business impact:** The system captures faces and head poses in videos. If a student's face is processed without proper consent, the liability risk is significant. Illinois BIPA has led to class-action settlements in the hundreds of millions.

---

## 6. Scaling — What Breaks at 10 Customers vs 1,000

### 6.1 What Breaks at 10 Institutions

| What | Why |
|---|---|
| Storage | 10 institutions × 50 videos × 200 MB = 100 GB. EBS is 50 GB. ❌ |
| GPU queue | All 10 institutions share 1 worker. If all upload Monday morning, last video waits hours |
| Email (Resend free tier) | 3,000/mo limit. 10 institutions × 30 users each = 300 users × ~2.5 emails/mo = 750 emails. ✅ OK |
| Support | 10 institutions × 2 support tickets/mo = 20 tickets. Email is fine. ✅ OK |

**Conclusion:** At 10 customers, storage is the first bottleneck. Upgrading EBS or adding S3 buys another 6-12 months.

### 6.2 What Breaks at 50 Institutions

| What | Why |
|---|---|
| Database | db.t3.micro handles ~60 connections. 50 institutions × concurrent users = ~150 concurrent → connection pool exhaustion ❌ |
| Analysis throughput | 1 GPU × ~5 min/video. 50 institutions × 2 videos/week = 100 videos. 500 min of GPU = 8.3 hours. Within a day ✅ but high variance days ❌ |
| Single-region latency | Institutions outside US-east get 100-300ms extra latency. Acceptable for upload but poor for dashboard loading ❌ |
| Support | 50 institutions × 5 tickets/mo = 250 tickets. Need dedicated support person ❌ |
| Cost | ~$1,500/mo infra + $3,000/mo support person = $4,500/mo. At $29/mo × 50 = $1,450 → losing $3,000/mo ❌ |

**Conclusion:** At 50 customers, the unit economics break unless price is $99/mo+. Even then, margins are thin because of the support headcount.

### 6.3 What Breaks at 200+ Institutions

| What | Why |
|---|---|
| Everything | Requires: auto-scaling, multi-region, dedicated GPU clusters, 24/7 support, compliance certifications, enterprise sales team, legal team, customer success team. This is a $1M+/yr operation |

**Conclusion:** This product cannot reach 200 institutions without venture capital or a strategic university partnership that funds the infrastructure build-out.

---

## 7. Summary: The Real Business Limitations

| # | Limitation | Severity | Cost to Fix |
|---|---|---|---|
| 1 | No LMS integration (LTI) | 🔴 Blocks all enterprise sales | ~$5,000/yr certification + 2 months dev |
| 2 | No SSO (SAML/OIDC) | 🔴 Blocks IT approval | ~2 weeks dev + identity provider costs |
| 3 | No pricing defined | 🔴 Can't sell | 1 day of spreadsheet work |
| 4 | No legal docs (ToS, DPA, Privacy) | 🔴 Can't sign contracts | ~$3,000-10,000 in legal fees |
| 5 | No billing system (Stripe) | 🔴 Can't collect money | ~2 weeks dev |
| 6 | No GDPR/CCPA compliance | 🟠 Blocks EU + California market | ~$5,000-20,000 legal + dev |
| 7 | No WCAG accessibility | 🟠 Blocks public US universities | ~2 weeks dev + audit |
| 8 | No support infrastructure | 🟠 Blocks institutions over 500 students | ~$2,000/mo for basic help desk |
| 9 | No differentiation vs competitors | 🟠 Unclear why anyone would switch | Strategic pivot needed |
| 10 | Unit economics unproven | 🟠 Cannot set price or forecast burn | 1 month of usage data collection |

### The Cold Truth

This is a **feature**, not a **product**. YOLO-based video analysis for proctoring is a single module inside what every competitor already ships as a complete platform. To sell this as a standalone SaaS:

- Need **$30,000-50,000** upfront for compliance, legal, and infrastructure
- Need **3-6 months** of integration development (LTI, SSO, billing)
- Need a **clear go-to-market angle** (Middle East focus? Open-source core? Privacy-first?)

**Without these, the project is a graduation demo — not a business.** Which is fine: that's exactly what it was built for.
