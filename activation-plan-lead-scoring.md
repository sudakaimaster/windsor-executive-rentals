# Activation Plan — Predictive Lead Scoring (MM IES)

PRD:

**Purpose:** Today, 80% of IES inbound leads wait 24-72 hours to be scored and routed—by which time buyer intent has cooled. Sales wastes time on low-quality leads, high-potential prospects go cold, and the majority of our highest-intent leads can't be scored on the most important pillar.

---

## 1. Feature Overview

| Field | Details |
|---|---|
| Feature/Capability Name | MM IES Predictive Lead Scoring |
| Platform Team Owner | |
| GTM Activation Owner | |
| Target Ship Date | APR 30 – May TBD |
| Target Activation Date | TBD |

---

## 2. Scoring Model Summary (NEW)

The new predictive lead scoring model replaces the legacy 3-pillar system with a 4-pillar approach. The legacy model has lost differentiation—93% of leads reach Salesforce with a "perfect" score of 100, and mid-range scores have a 90%+ disqualification rate.

### 4-Pillar Scoring Model

| Pillar | Weight | Description |
|---|---|---|
| ICP Fit | 50% | Ideal Customer Profile match based on firmographic and technographic data (sourced via ZoomInfo enrichment) |
| Engagement | TBD% | Behavioral signals: web visits, content downloads, email engagement, event attendance |
| Intent | TBD% | Third-party intent signals including Surge multi-entity flags |
| Product Usage (NEW) | TBD% | 21 first-party signals from Desktop, QBO, and Payroll product usage data |

### Scoring Tiers

| Tier | Score Range | Expected Action |
|---|---|---|
| A (Hot) | TBD | Immediate hand-raiser routing; <10 min SLA |
| B (Warm) | TBD | Standard BDR follow-up within 24 hours |
| C (Cold) | TBD | Nurture track; no immediate outbound |

### Example Scoring Scenarios

| Scenario | ICP | Engagement | Intent | Product Usage | Total | Tier |
|---|---|---|---|---|---|---|
| QBO Outgrower with high engagement | High | High | Medium | High | TBD | A |
| Net-new prospect, low fit, clicked 1 email | Low | Low | None | N/A | TBD | C |
| Desktop Refugee, strong intent signals | High | Medium | High | Medium | TBD | A |
| Good ICP, no engagement yet | High | None | None | N/A | TBD | C |

> **Action Item:** Finalize pillar weights and tier boundaries. Get sign-off from Sales + Marketing leadership by [DATE].

### MQL Threshold Definition (NEW — Critical)

The new MQL definition replaces legacy scoring logic. A lead qualifies as MQL when:

- **ICP Fit score ≥ [THRESHOLD]** (mandatory — no lead can MQL without strong ICP fit)
- **Combined Unified Score ≥ [THRESHOLD]** (across all 4 pillars)
- **Additional criteria for existing customers:** [TBD — factor in segment and product usage]

> **Action Item:** Define exact thresholds and get cross-functional sign-off by 3/20.

---

## 3. System Architecture (NEW)

```
Lead Source (Web Form / Event / Content)
        │
        ▼
Adobe Experience Platform (AEP)
   [Streaming mode — 10 min refresh]
        │
        ▼
┌───────────────────────────────────┐
│     Relevvo Scoring Engine        │
│  ┌─────────┐  ┌───────────────┐  │
│  │ ICP Fit  │  │  Engagement   │  │
│  │  (50%)   │  │    (TBD%)     │  │
│  └─────────┘  └───────────────┘  │
│  ┌─────────┐  ┌───────────────┐  │
│  │  Intent  │  │ Product Usage │  │
│  │  (TBD%)  │  │    (TBD%)     │  │
│  └─────────┘  └───────────────┘  │
│                                   │
│  ZoomInfo Enrichment (ICP)        │
│  Surge Signals (Intent)           │
│  Data Lake — 21 1P Signals (Usage)│
└───────────────────────────────────┘
        │
        ▼
    Amazon S3
   [Tiered tagging: Hot / Warm / Cold]
        │
        ├──────────────┐
        ▼              ▼
    Marketo          SFDC
  (MQL routing)   (BDR tools, reports)
```

---

## 4. Business Context

### What business problem does this solve?

Today, 80% of IES inbound leads wait 24-72 hours to be scored and routed—by which time buyer intent has cooled. Our current model has lost differentiation: 93% of leads reach Salesforce with a "perfect" score of 100, and mid-range scores have a 90%+ disqualification rate. Sales wastes time on low-quality leads while high-potential prospects go cold. With ICP weighted at 50% in the new model, no lead can MQL without strong ICP fit, enabling Sales to prioritize high-quality leads and improve conversion.

### Who benefits from this capability?

| Beneficiary | How They Benefit |
|---|---|
| Sales Reps | Prioritize high-quality leads with strong ICP fit. Reduced time wasted on low-quality leads. Clear tiering (A/B/C) with actionable routing. Existing customer segments (Desktop Refugee, QBO Outgrower) enable tailored conversations. |
| Marketing | Faster lead scoring (<10 min vs. 24-72 hours) preserves buyer intent. Differentiated scoring replaces the current model where 93% of leads score 100. Enables score-aware nurture campaigns by tier and customer segment. |
| Marketing Ops | Clearer MQL definition tied to ICP + Unified threshold. Automated suppression of non-ICP leads reduces noise. Score-based audience segmentation enables precision targeting. |
| Revenue Ops | Real-time scoring enables faster routing and SLA tracking. Data-driven P1-P4 prioritization replaces campaign-based logic. Score history pipeline enables trend analysis and model accountability. |

---

## 5. RACI Matrix (NEW)

| Workstream | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Platform / Scoring Engine | | | | |
| Marketing Ops (Marketo, MQL) | | | | |
| Sales Innovation / Data Engineering | | | | |
| Sales Enablement | | | | |
| GTM Tech (Monitoring, Dashboards) | | | | |
| Revenue Ops | | | | |

> **Action Item:** Fill in names for every cell. Every line item in the execution plan below should map to one of these owners.

---

## 6. Pre-Ship Execution Plan

**Prod Go-Live Target: 4/30**

### 6.1 Platform (Tracked in Jira)

| Milestone | What Needs to Happen | Dependencies | Due | Target Date | Owner | Status |
|---|---|---|---|---|---|---|
| Pre-Score 470K IES Customer Accounts | Load all IES prospect/customer accounts to Relevvo; generate scores; store in accessible location | — | Week before go-live | 4/20 | | ⬜ |
| Product Usage Pipeline | Build Data Lake → Scoring pipeline for 21 1P signals | DE-P3 (data access) | Before go-live | 4/20 | | ⬜ |
| Existing Customer Flag | Implement Relevvo_Is_Existing_Customer detection logic | — | Before go-live | 4/20 | | ⬜ |
| Segment Assignment Logic | Implement Relevvo_Segment field (Desktop Refugee, QBO Outgrower, etc.) | Existing Customer Flag | Before go-live | 4/20 | | ⬜ |
| Tiered Routing Rules | Configure S3 tagging for hand-raisers, hot leads, cold leads | — | Before go-live | 4/20 | | ⬜ |
| AEP Streaming Configuration | Ensure AEP → S3 dataflows operate in streaming mode (10 min) | — | Before go-live | 4/20 | | ⬜ |
| ZoomInfo Fallback Logic | Define ICP scoring behavior when enrichment fails | — | Before go-live | 4/20 | | ⬜ |

### 6.2 Marketing Ops

| # | Item | What Needs to Happen | Dependencies | Due | POC | Start | ETA | Status |
|---|---|---|---|---|---|---|---|---|
| MO-P1 | Marketo Custom Fields (9 fields) | Create all new score fields in Marketo | — | 1 week before go-live | | | | 🔴 Not Started |
| MO-P2 | Hot Lead Definition | Define criteria for "Hot Lead" tag (engagement threshold) | MQL threshold decision | Before go-live | | | | 🔴 Not Started |
| MO-P3 | Load Pre-Scored Accounts to Marketo | Ingest 600K pre-scored account data into Marketo for matching | Platform: Pre-Score 470K accounts | Before go-live | | | | 🔴 Not Started |
| MO-P4 | Updated MQL Definition | Replace legacy MQL logic with ICP + Unified threshold | Cross-functional sign-off | Before go-live | | Mar 10 | | 🔴 Not Started |
| MO-P5 | ICP Suppression Rules | Prevent routing of non-ICP leads | MO-P4 (MQL definition) | Before go-live | | | | 🔴 Not Started |
| MO-P6 | Internal Training / Communication | Training on new scoring model and changes to scoring workflows | All MO-P items above | Before go-live | | | | 🔴 Not Started |
| MO-P7 | Go-Live Cutover Plan | Document legacy vs. new scoring steps, cutover timing, rollback triggers | All items | Before go-live | | | | 🔴 Not Started |

### 6.3 Sales Innovation / Data Engineering

| # | Item | What Needs to Happen | Dependencies | Due | POC | ETA | Status |
|---|---|---|---|---|---|---|---|
| DE-P1 | Data Lake → SFDC Sync | Add new score fields to daily sync; target <2 min real-time | Platform fields finalized | Before go-live | | | 🔴 Not Started |
| DE-P2 | Multi-Entity Signal Access | Expose Surge multi-entity flags to scoring pipeline | — | Before go-live | | | 🟡 In Progress |
| DE-P3 | Product Usage Data Access | Expose Desktop, QBO, Payroll signals to scoring pipeline | — | Before go-live | | | 🟡 In Progress |
| DE-P4 | Baseline Snapshot | Capture baseline scoring & conversion metrics before cutover | — | Before go-live (CRITICAL — cannot slip) | | | 🔴 Not Started |

---

## 7. Rollout Strategy: Phased Approach (NEW)

Rather than a big-bang cutover, a phased rollout reduces risk and generates real conversion data for validation.

### Recommended Phases

| Phase | Duration | What Happens | Scoring Model in Effect |
|---|---|---|---|
| **Shadow Mode** | Week 1-2 (4/30 – 5/13) | New scores are calculated and written to SFDC alongside legacy scores. Legacy scoring still drives routing and MQL decisions. BDRs can see new scores but are not required to act on them. | Legacy (primary) + Relevvo (shadow) |
| **A/B Split** | Week 3 (5/14 – 5/20) | 50% of incoming leads scored and routed by new model, 50% by legacy. Enables direct conversion comparison. | 50/50 split |
| **Full Cutover** | Week 4+ (5/21+) | If validation passes decision gates, all leads scored by new model. Legacy scoring retired. | Relevvo (primary) |

### Decision Gate: Shadow → A/B Split
- New model score distribution is healthy (<30% at max score)
- Latency SLAs are met (hand-raisers <10 min)
- No critical bugs or data pipeline failures
- BDR qualitative feedback is neutral or positive

### Decision Gate: A/B Split → Full Cutover
- New model leads convert at ≥ parity with legacy leads (MQL → SAL)
- No significant volume drop (MQL volume within 20% of baseline)
- P1-P4 switch decision validated (SI-5)

---

## 8. Rollback Plan (NEW)

### Rollback Triggers

If any of the following occur, initiate rollback to legacy scoring:

| Trigger | Threshold | Detection Method |
|---|---|---|
| MQL volume drop | >30% below baseline in first 48 hours | GT-4 monitoring dashboard |
| Hand-raiser latency | >30 min average for 4+ consecutive hours | GT-3 latency monitoring |
| Score distribution collapse | >50% of leads in a single tier | GT-4 health check |
| Data pipeline failure | AEP → S3 or S3 → Marketo/SFDC sync down >1 hour | Pipeline alerting |
| ZoomInfo enrichment failure | >40% of leads fail enrichment (ICP pillar = 50% weight) | Enrichment monitoring |
| Critical BDR escalation | 5+ BDRs report unusable scores in first 24 hours | SE feedback channel |

### Rollback Steps

1. **Marketo:** Revert MQL smart campaign to legacy scoring logic (documented in MO-P7 cutover plan)
2. **SFDC:** Switch lead assignment rules back to legacy priority field logic
3. **S3 Routing:** Disable Relevvo-based tiered tagging; re-enable legacy routing tags
4. **Communication:** Notify Sales Enablement → BDR managers → BDRs within 2 hours of rollback decision
5. **Post-Mortem:** Conduct root cause analysis within 48 hours; document in this plan

### Rollback Owners

| System | Primary Owner | Backup |
|---|---|---|
| Marketo | | |
| SFDC | | |
| S3 / Pipeline | | |
| Communication | | |

---

## 9. Risk Register (NEW)

| # | Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R1 | ZoomInfo enrichment failure rate spikes — ICP pillar (50% weight) cannot score | Medium | High | ZoomInfo Fallback Logic (pre-ship item); define acceptable failure rate threshold (<10%); degrade gracefully to 3-pillar scoring if enrichment unavailable | |
| R2 | Product Usage pipeline not ready by go-live | Medium | High | Decouple launch: go live with 3 pillars (ICP + Engagement + Intent), add Product Usage in Week 2-3 as fast-follow | |
| R3 | BDR resistance to new scoring model | High | Medium | Score Skeptic Playbook (SE-6); parallel validation proof points; BDR champions program; Score Clinic sessions | |
| R4 | AEP streaming latency exceeds 10 min target | Low | High | Alert at 15 min; fallback to batch scoring every 30 min; escalation to Platform team | |
| R5 | Score distribution still clusters at high end (loss of differentiation) | Medium | High | GT-4 daily health checks; model retraining (GT-9) at Week 5; threshold tuning cycles (SI-7, SI-8) | |
| R6 | MQL volume drops significantly, creating pipeline gap | Medium | High | Phased rollout (shadow mode first); adjustable thresholds; rollback plan with 48-hour trigger | |
| R7 | Pre-scored account data is stale or incomplete | Low | Medium | Validate sample of pre-scored accounts before go-live; establish refresh cadence | |

---

## 10. Post-Ship Execution Plan

### 10.1 Activation Timeline

| Milestone | Target Date | Owner | Status |
|---|---|---|---|
| Feature ships to production | 4/30 | Platform | ⬜ |
| GTM team trained/enabled | | | ⬜ |
| Process documentation updated | | | ⬜ |
| Tool configuration complete | | | ⬜ |
| First campaign/use case live | | | ⬜ |
| Week 1 Checkpoint — Post Release Review | Week of May 5 | | ⬜ |
| 30-day adoption checkpoint | | | ⬜ |
| 90-day impact review | | | ⬜ |

### 10.2 Launch Phase (Weeks 1-2)

**GTM Tech**

| # | Item | What Needs to Happen | Week | POC | ETA | Status |
|---|---|---|---|---|---|---|
| GT-1 | Scoring v2 Go-Live | Deploy 4-pillar scoring to production | Week 1 | | 4/30 | Pending |
| GT-2 | Score Monitoring & Alerting | Configure alerts: distribution, latency, processing success | Week 1 | | 4/30 | |
| GT-3 | Latency Monitoring by Lead Type | Track end-to-end latency separately for hand-raisers, hot, cold | Week 1 | | 4/30 | |
| GT-4 | Score Distribution Health Check | Daily check: <30% at max score; alert if >40% in any tier | Week 1+ | | 4/30 | |
| GT-5 | Product Usage Signal Validation | Verify 42 signals firing correctly for existing customers | Week 1 | | 4/30 | |

**Sales Enablement**

| # | Item | What Needs to Happen | Week | Status |
|---|---|---|---|---|
| SE-1 | "What Changed v2" Communication | One-pager: 3-pillar → 4-pillar; new latency tiers; existing customer handling (see Appendix A) | Week 1 | |
| SE-2 | Score Adoption Training v2 | Train on 4 pillars, A/B/C tiers, Relevvo_Top_Drivers, existing customer segments | Week 1 | |
| SE-3 | Existing Customer Segment Training | Train BDRs on interpreting Desktop Refugee, QBO Outgrower, NTTF segments | Week 1 | |
| SE-4 | Score Feedback Process Training | Train on weekly feedback form including Product Usage accuracy | Week 1 | |
| SE-NEW1 | BDR Champions Program | Designate 2-3 BDRs as beta testers during shadow mode; they become peer advocates at full cutover | Week 1 | |
| SE-NEW2 | Score Clinic Session | Live session where BDRs bring real leads and walk through how the new model scored them | Week 2 | |

**Sales Innovation**

| # | Item | What Needs to Happen | Week | Status |
|---|---|---|---|---|
| SI-1 | Parallel Validation v2 | Run old scoring vs. 4-pillar Relevvo for 2 weeks | Weeks 1-2 | |
| SI-2 | Score in Reports & List Views | Add all Relevvo fields to BDR tools | Week 1 | |
| SI-3 | Existing Customer Reports | Create SFDC reports filtering by Relevvo_Is_Existing_Customer and segment | Week 1 | |

**Marketing Ops**

| # | Item | What Needs to Happen | Week | Status |
|---|---|---|---|---|
| MO-1 | Score-Based Audience Definitions v2 | Create Marketo audiences by tier AND by existing customer status | Week 1 | |
| MO-2 | Existing Customer Campaign Targeting | Segment campaigns by Desktop Refugee, QBO Outgrower, NTTF | Week 2 | |
| MO-3 | MQL Conversion Monitoring | Validate MQL → SAL performance under new logic | Week 2 | |

### 10.3 Validation Phase (Weeks 3-4)

**Decision Gate: Launch → Validation** (NEW)
- Parallel validation shows new model is not degrading conversion
- Latency SLAs met for all lead types
- Score distribution is healthy and differentiated
- No rollback triggers activated

**Sales Innovation**

| # | Item | What Needs to Happen | Week | Status |
|---|---|---|---|---|
| SI-4 | Parallel Validation Results Review | Analyze: conversion by tier, latency by lead type, product usage accuracy | Week 3 | |
| SI-5 | P1-P4 Logic Switch Decision | Based on validation: switch from campaign-based to score-based | Week 3 | |
| SI-6 | P1-P4 Configuration Switch | If validated, configure SFDC priority field logic | Week 3-4 | |
| SI-7 | Score Threshold Tuning Cycle 1 | Adjust P-level boundaries based on actual conversion data | Week 4 | |

**Sales Enablement**

| # | Item | What Needs to Happen | Week | Status |
|---|---|---|---|---|
| SE-5 | P1-P4 Workflow Training | Train BDRs on exhaustion model with 4-pillar scoring | Week 3-4 | |
| SE-6 | Score Skeptic Playbook v2 | Update with parallel validation results; include Product Usage value prop | Week 3 | |

**GTM Tech**

| # | Item | What Needs to Happen | Week | POC | ETA | Status |
|---|---|---|---|---|---|---|
| GT-6 | Score History Pipeline | Daily sync of scores + history + change reasons to Data Lake | Week 3 | | | |
| GT-7 | Scoring Accuracy Dashboard | Build: predicted tier vs. actual conversion; conversion by segment | Week 4 | | | |
| GT-8 | R² Correlation Baseline | Calculate initial R² (Score → MQL → SAL CVR); target >0.9 | Week 4 | | | |

### 10.4 Optimization Phase (Weeks 5-8)

**Decision Gate: Validation → Optimization** (NEW)
- R² > 0.7 (with target of 0.9 by Week 8)
- P1-P4 switch decision made and executed
- Conversion by tier shows meaningful differentiation (A-tier converts ≥2x C-tier)
- Leadership sign-off to proceed

**GTM Tech**

| # | Item | What Needs to Happen | Week | POC | ETA | Status |
|---|---|---|---|---|---|---|
| GT-9 | First Monthly Model Retraining | Recalculate weights with 30 days of conversion data | Week 5 | | | |
| GT-10 | Model Performance Assessment | AUC, Precision, Lift, R² vs. targets | Week 5 | | | |
| GT-11 | Product Usage Signal Tuning | Adjust weights based on which 1P signals correlate with conversion | Week 6 | | | |

**Sales Innovation**

| # | Item | What Needs to Happen | Week | POC | ETA | Status |
|---|---|---|---|---|---|---|
| SI-8 | Score Threshold Tuning Cycle 2 | Second adjustment of P-level boundaries | Week 6 | TBD | | |
| SI-9 | Scoring Accuracy Report to Leadership | Present: conversion improvement, adoption metrics, model performance | Week 8 | TBD | | |

**Marketing Nurture** (Plan in progress)

| # | Item | What Needs to Happen | Week | POC | ETA | Status |
|---|---|---|---|---|---|---|
| MO-3 | Score-Aware Nurture Branching | Configure nurture streams by tier + existing customer status | Week 5-6 | | | |
| MO-4 | Existing Customer Upsell Campaigns | Launch targeted campaigns for Desktop Refugee, QBO Outgrower segments | Week 6+ | | | |

---

## 11. Communication Plan (NEW)

| Audience | Message | Channel | Timing | Owner |
|---|---|---|---|---|
| MM Sales Leadership | Executive summary: why we're changing, expected impact on pipeline, what success looks like | Deck + meeting | 2 weeks pre-launch (4/16) | |
| BDR Managers | Detailed walkthrough: new scoring model, tiering, routing changes, enablement plan, their role in rollout | Training session | 1 week pre-launch (4/23) | |
| BDRs | "What's Changing" one-pager + hands-on training (see SE-1, SE-2) | Email + live session | Launch week (4/30) | |
| Marketing Leadership | New MQL definition, expected volume changes (may decrease short-term), audience segmentation opportunities | Deck + meeting | 2 weeks pre-launch (4/16) | |
| Revenue Ops | Field changes, SFDC sync logic, reporting updates, dashboard requirements | Technical doc + meeting | 3 weeks pre-launch (4/9) | |
| Executive Sponsors | High-level progress updates at each phase gate | Email summary | Weekly during Weeks 1-4; biweekly after | |

---

## 12. Sales Enablement Content Plan (NEW)

### 12.1 "What Changed" One-Pager (SE-1)

Key content to include:

**Before (Legacy Model)**
- 3-pillar scoring (ICP + Engagement + Intent)
- 93% of leads arrive at SFDC with a score of 100
- 24-72 hour scoring latency
- Mid-range scores have 90%+ disqualification rate
- No product usage data factored in

**After (Predictive Lead Scoring v2)**
- 4-pillar scoring (ICP + Engagement + Intent + Product Usage)
- Differentiated tier distribution (A / B / C)
- <10 min scoring latency for hand-raisers
- ICP weighted at 50% — strong ICP fit required for MQL
- Existing customer intelligence: segment labels (Desktop Refugee, QBO Outgrower, NTTF)
- Relevvo_Top_Drivers field shows why a lead scored the way it did

### 12.2 BDR FAQ (NEW)

| Question | Answer |
|---|---|
| **Why did my lead volume drop?** | The new model applies stricter ICP filtering. You'll get fewer leads, but they'll be significantly higher quality. Focus on A-tier leads first. |
| **What do the customer segments mean?** | **Desktop Refugee:** Currently on Desktop, likely evaluating cloud migration. **QBO Outgrower:** QBO customer showing signs of needing more capability. **NTTF:** Net-new to the franchise — no existing Intuit product relationship. |
| **How do I read Relevvo_Top_Drivers?** | This field shows the top 3 factors that drove the lead's score. Use this to personalize your outreach — e.g., if "Product Usage: Desktop" is a top driver, reference their Desktop experience. |
| **What if I think a score is wrong?** | Use the weekly feedback form (SE-4 training). Your input directly improves the model during tuning cycles. |
| **When will scores appear in SFDC?** | Hand-raisers: <10 min. Hot leads: <30 min. Cold leads: scored in batch. |
| **Is this replacing P1-P4 priorities?** | Not immediately. During Weeks 1-2, legacy P1-P4 still drives routing. After validation (Week 3-4), we'll transition to score-based prioritization. |

---

## 13. Adoption Tracking

| Metric | 30-Day Target | 90-Day Target | Who Measures |
|---|---|---|---|
| % of BDRs actively viewing new score fields in SFDC | 80% | 95% | |
| % of leads scored by new model (vs. legacy fallback) | 90% | 99% | |
| Avg lead-to-first-touch latency (hand-raisers) | <10 min | <5 min | |
| Avg lead-to-first-touch latency (all leads) | <2 hours | <1 hour | |
| BDR feedback form submission rate | 50%+ weekly | 30%+ weekly | |
| Score distribution health (% at max score) | <30% | <20% | |

---

## 14. Business Impact Measurement

| Outcome Metric | Baseline (Pre-Launch) | 30-Day Target | 30-Day Actual | 90-Day Target | 90-Day Actual | Notes |
|---|---|---|---|---|---|---|
| MQL → SAL conversion rate | _Capture in DE-P4_ | +15% vs. baseline | | +25% vs. baseline | | |
| A-tier lead conversion rate | N/A (new metric) | Establish baseline | | 2x C-tier rate | | |
| Lead scoring latency (hand-raisers) | 24-72 hours | <10 min | | <5 min | | |
| % of leads at max score (100) | 93% | <30% | | <20% | | |
| Mid-range score disqualification rate | 90%+ | <60% | | <40% | | |
| Sales-accepted lead (SAL) volume | _Capture in DE-P4_ | ≥ parity | | +10% vs. baseline | | |
| Pipeline generated from scored leads | _Capture in DE-P4_ | Track | | +15% vs. baseline | | |
| Model R² (Score → Conversion) | N/A | >0.7 | | >0.9 | | |
| BDR time-to-work (first action on lead) | _Capture in DE-P4_ | -30% | | -50% | | |

---

## 15. Next Steps — Priority Action Items (NEW)

| # | Action Item | Due Date | Owner | Status |
|---|---|---|---|---|
| 1 | Assign owners to every line item (especially MO-P1 through MO-P7 and all RACI cells) | 3/14 | | |
| 2 | Define new MQL threshold (MO-P4) and get cross-functional sign-off from Sales + Marketing leadership | 3/20 | | |
| 3 | Fill in Adoption Tracking and Business Impact tables with concrete targets | 3/20 | | |
| 4 | Write the detailed rollback plan with explicit triggers and steps (MO-P7) | 3/20 | | |
| 5 | Decide on phased rollout (shadow → A/B → full) vs. big-bang and adjust timeline | 3/27 | | |
| 6 | Begin Sales Enablement content development (one-pager, FAQ, training deck) | 3/27 | | |
| 7 | Capture baseline metrics — DE-P4 (CRITICAL: cannot slip, needed for all post-launch measurement) | 4/7 | | |
| 8 | Complete all Marketing Ops pre-ship items (MO-P1 through MO-P7) | 4/15 | | |
| 9 | Communicate to MM Sales Leadership and Marketing Leadership (Communication Plan) | 4/16 | | |
| 10 | Train BDR Managers (Communication Plan) | 4/23 | | |
| 11 | Go-live: begin shadow/parallel scoring period | 4/30 | | |
| 12 | Week 1 Checkpoint — Post Release Review | Week of May 5 | | |

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| ICP | Ideal Customer Profile — firmographic and technographic fit score |
| MQL | Marketing Qualified Lead — meets threshold for sales handoff |
| SAL | Sales Accepted Lead — MQL that sales confirms as worth pursuing |
| BDR | Business Development Representative |
| Relevvo | Predictive scoring engine used for the new 4-pillar model |
| Desktop Refugee | Existing customer segment: currently on Desktop product, evaluating cloud migration |
| QBO Outgrower | Existing customer segment: QBO customer showing signs of needing more capability |
| NTTF | Net New to the Franchise — no existing Intuit product relationship |
| 1P Signals | First-party signals from product usage data |
| Surge | Third-party intent data provider (multi-entity flags) |
| AEP | Adobe Experience Platform — data streaming infrastructure |
| P1-P4 | Lead priority tiers in SFDC (P1 = highest priority) |
| R² | R-squared — statistical measure of how well scores predict actual conversion |
