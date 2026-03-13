# BRD-1: Marketo Integration — Accountant Lead Management Pipeline

| Field | Value |
|---|---|
| **Document Owner** | *(TBD)* |
| **Last Updated** | *(TBD)* |
| **Release Status** | Draft |
| **Initiative ID** | BRD-ACCT-MKT-001 |
| **Dependency** | None (foundational) |
| **Downstream Dependent** | BRD-ACCT-ABM-001 (ABM Platform Integration) |

---

## 1. Roadmap Prioritization Approvers

| Functional Area | Name | BRD Approval Status |
|---|---|---|
| Marketing | *(TBD)* | Pending |
| GTM Tech | *(TBD)* | Pending |
| Sales Tech | *(TBD)* | Pending |
| Sales Innovation | *(TBD)* | Pending |

## 2. Leadership Alignment

| Functional Area | Name | Alignment |
|---|---|---|
| Marketing | *(TBD)* | Pending |
| Sales | *(TBD)* | Pending |

---

## 3. Executive Summary

This BRD defines the technical requirements to integrate accountant personas into the Marketo lead management pipeline. The accountant channel is forecast to deliver 3,471 IES contracts in 2H (a 24x scale-up from 1H's 141 contracts). Without automation of accountant ingestion, enrichment, scoring, and routing in Marketo, this forecast is at serious risk.

**Current State (As-Is):**
- Marketo does not ingest, identify, or score accountant leads.
- Accountant leads arriving via IPD abandonment, webinars, or content engagement enter a "black hole" — they are not scored, not routed, and not actionable by Sales.
- Manual enrichment processes ("fire drills") are the only mechanism today, which cannot scale.
- Lead-to-Account (L2A) matching logic maps all leads to Client Accounts; accountants should map to their Firm Account (employer), not the end-user client.

**Future State (To-Be):**
- Marketo ingests accountant leads from all upstream sources (IPD, webinars, content engagement, ABM handoffs).
- Automated enrichment via the 1P BPP pipeline identifies accountant persona type and firm type at ingestion time.
- L2A logic correctly resolves accountant leads to Firm Accounts in SFDC.
- Scoring model evaluates accountant-specific engagement signals and triggers lead pass to Sales at threshold.
- Nurture campaigns operate at scale with proper segmentation by firm tier and engagement stage.

**Estimated Business Impact:** 350–400 incremental IES contracts (~$6.65M–$7.6M revenue).

---

## 4. Scope

### 4.1 In Scope

| # | Capability | Description |
|---|---|---|
| 1 | Accountant Identification | Add a `is_accountant` (boolean) and `accountant_persona_type` (enum) field to the Marketo Person object schema to distinguish accountant leads from standard MM prospects. |
| 2 | 1P BPP Enrichment Extension | Extend the first-party Business Profile Provider (BPP) enrichment pipeline to automatically classify accountant personas and firm types upon lead ingestion. |
| 3 | L2A Logic Modification | Update Lead-to-Account matching logic so accountant Marketo Persons resolve to the **Firm Account** (employer entity) rather than the Client Account (end-user). |
| 4 | Accountant Scoring Model | Implement a scoring model for accountant-specific engagement signals: webinar attendance, IPD abandonment, email engagement, content downloads. |
| 5 | Firm Tier Segmentation | Classify accountant leads by firm segment: **National**, **HVAM**, or **Unmanaged** — eliminating manual Lead Repo exports. Map each lead to the assigned account owner in SFDC. |
| 6 | Automated Lead Pass | Trigger MQL handoff to Sales (via SFDC sync) when composite lead score crosses a configurable threshold. |
| 7 | Nurture Campaign Orchestration | Enable automated nurture flows segmented by engagement stage (registered/attended, registered/not-attended, hand-raisers, post-meeting non-buyers, stalled leads). |
| 8 | IPD Abandoner Ingestion | Ingest IES IPD abandoner events into Marketo, identify which are accountants, score accordingly. |

### 4.2 Out of Scope

| # | Item | Rationale |
|---|---|---|
| 1 | ABM platform procurement, configuration, or signal ingestion | Covered in BRD-ACCT-ABM-001 |
| 2 | DSP/LinkedIn ad campaign creation or management | Marketing Ops responsibility |
| 3 | SFDC schema changes beyond L2A field mapping | Requires separate Sales Tech BRD |

---

## 5. Technical Requirements

### 5.1 Marketo Schema Changes

| Requirement ID | Description | Object | Field(s) | Type | Notes |
|---|---|---|---|---|---|
| MKT-SCH-001 | Add accountant identifier flag | Person | `is_accountant` | Boolean | Default `false`; set by enrichment pipeline or manual override |
| MKT-SCH-002 | Add accountant persona type | Person | `accountant_persona_type` | Picklist: `Partner`, `Staff`, `Decision-Maker`, `Unknown` | Populated by BPP enrichment |
| MKT-SCH-003 | Add firm tier classification | Person | `firm_tier` | Picklist: `National`, `HVAM`, `Unmanaged` | Derived from Firm Account segment in SFDC |
| MKT-SCH-004 | Add firm account SFDC ID | Person | `firm_account_id__c` | String (18-char SFDC ID) | Foreign key to Firm Account in SFDC |

### 5.2 Enrichment Pipeline (1P BPP Extension)

| Requirement ID | Description | Details |
|---|---|---|
| MKT-ENR-001 | Extend BPP ingestion trigger | On new lead creation in Marketo, invoke BPP enrichment API to classify: (a) whether the lead is an accountant persona, (b) firm type, (c) firm tier. |
| MKT-ENR-002 | Enrichment data mapping | Map BPP response fields → Marketo Person fields: `is_accountant`, `accountant_persona_type`, `firm_tier`, `firm_account_id__c`. |
| MKT-ENR-003 | Enrichment SLA | BPP enrichment must complete within **60 seconds** of lead creation to avoid scoring delays. Define a retry policy (max 3 retries, exponential backoff). |
| MKT-ENR-004 | Enrichment failure handling | If BPP enrichment fails after retries, flag lead with `enrichment_status = 'failed'` and route to manual enrichment queue. Alert GTM Tech via Slack webhook. |

### 5.3 Lead-to-Account (L2A) Logic

| Requirement ID | Description | Details |
|---|---|---|
| MKT-L2A-001 | Conditional L2A routing | If `is_accountant = true`, match to Firm Account (employer) in SFDC using firm name + domain fuzzy match. If `is_accountant = false`, use existing Client Account matching logic. |
| MKT-L2A-002 | Firm Account resolution | Use a weighted matching algorithm: (1) exact domain match (weight 1.0), (2) firm name Levenshtein distance < 3 (weight 0.7), (3) BPP-provided firm ID direct lookup (weight 1.0). Highest-confidence match wins. |
| MKT-L2A-003 | Unresolved lead handling | If no Firm Account match is found with confidence > 0.7, route lead to an "Unmatched Accountant" holding queue for manual review. |
| MKT-L2A-004 | SFDC owner assignment | After L2A resolution, stamp the `lead_owner` field with the Firm Account's assigned owner from SFDC. |

### 5.4 Scoring Model

| Requirement ID | Description | Details |
|---|---|---|
| MKT-SCR-001 | Behavioral scoring dimensions | Define scoring weights for: webinar registration (+5), webinar attendance (+15), IPD start (+10), IPD abandonment (+8), email open (+2), email click (+5), content download (+10), form fill (+12), hand-raiser action (+25). |
| MKT-SCR-002 | Demographic scoring dimensions | Firm tier weighting: National (+20), HVAM (+15), Unmanaged (+5). Persona type: Decision-Maker (+15), Partner (+10), Staff (+5). |
| MKT-SCR-003 | Score decay | Implement a time-based decay function: reduce behavioral score by 10% every 30 days of inactivity to prevent stale leads from triggering MQL. |
| MKT-SCR-004 | MQL threshold | Lead passes to Sales when composite score (behavioral + demographic) >= **100 points**. Threshold is configurable per firm tier. |
| MKT-SCR-005 | Score sync to SFDC | Sync `marketo_lead_score` and `score_breakdown` (JSON blob) to SFDC Lead/Contact record on every score change event. |

### 5.5 Automated Lead Pass & SFDC Sync

| Requirement ID | Description | Details |
|---|---|---|
| MKT-SYN-001 | MQL trigger | When score >= threshold, change lead status to `MQL`, set `mql_date`, and sync to SFDC. Create a Task in SFDC assigned to the account owner with subject "New Accountant MQL — Immediate Follow-up Required". |
| MKT-SYN-002 | Bi-directional sync | SFDC opportunity stage changes (e.g., `Meeting Scheduled`, `Closed Won`, `Closed Lost`) must sync back to Marketo to update lifecycle stage and suppress/adjust nurture eligibility. |
| MKT-SYN-003 | Sync latency SLA | Marketo → SFDC sync must complete within **5 minutes** of MQL event. SFDC → Marketo sync must complete within **15 minutes**. |

### 5.6 Nurture Campaign Architecture

| Requirement ID | Description | Details |
|---|---|---|
| MKT-NUR-001 | Segmentation streams | Implement nurture streams segmented by: (a) Webinar — Registered & Attended, (b) Webinar — Registered & Not Attended, (c) Hand-Raisers (immediate Sales outreach, no nurture), (d) Post-Meeting Non-Buyers, (e) Stalled/Cold Leads (re-engagement), (f) IPD Abandoners. |
| MKT-NUR-002 | Entry criteria | Each stream has defined entry Smart List criteria based on `is_accountant`, `accountant_persona_type`, `firm_tier`, engagement activity, and lifecycle stage. |
| MKT-NUR-003 | Cadence rules | Default cadence: 1 email per 7 days. Suppress if lead has open SFDC opportunity in stage >= `Meeting Scheduled`. Allow cadence override per campaign via Marketo program tokens. |
| MKT-NUR-004 | Exit criteria | Remove from nurture on: MQL (score threshold met), Closed Won, Unsubscribe, Hard Bounce, or manual Sales opt-out via SFDC checkbox. |

---

## 6. Use Case Detail: Mid-Market Webinar Flow

**Trigger:** Accountant registers for or attends an MM webinar promoted through TOF programmatic, paid social, and/or ABM campaigns.

**Processing Steps:**

1. **Ingestion** — Webinar platform (e.g., ON24, Zoom Webinars) sends registration/attendance event to Marketo via webhook or native connector.
2. **Enrichment** — BPP enrichment fires on lead create/update. Determines `is_accountant`, `accountant_persona_type`, `firm_tier`.
3. **L2A Resolution** — If `is_accountant = true`, match to Firm Account. Expand contact list: query BPP/SFDC for additional decision-makers within the same firm.
4. **Scoring** — Apply behavioral score (registration +5, attendance +15). Apply demographic score based on firm tier and persona type.
5. **Nurture Assignment** — Route to appropriate nurture stream based on attendance status.
6. **MQL Check** — If composite score >= threshold, trigger lead pass to Sales. Create SFDC Task.
7. **Post-Meeting Handling** — If Sales meeting occurs but no purchase, move to "Post-Meeting Non-Buyer" nurture stream after configurable cool-down period (default: 14 days).

---

## 7. Phased Delivery Plan

| Phase | Capability | Target Date | Dependencies | Acceptance Criteria |
|---|---|---|---|---|
| **Phase 1** | Accountant identification + Marketo schema changes + BPP enrichment extension | *(TBD)* | Accountant business rules sign-off; BPP API contract finalized | New leads correctly tagged with `is_accountant`, `firm_tier`; enrichment completes within 60s SLA for 95th percentile |
| **Phase 2** | Lead scoring model + L2A logic update + SFDC sync | *(TBD)* | Phase 1 complete | Scoring model produces expected scores for test leads; L2A correctly resolves to Firm Accounts; MQL triggers create SFDC Tasks |
| **Phase 3** | Nurture campaign orchestration + IPD abandoner ingestion | *(TBD)* | Phase 2 complete | All nurture streams operational; IPD abandoners ingested and scored; end-to-end flow validated |

---

## 8. Key Existing Marketo Campaigns (Reference)

| Campaign ID | Name | Purpose |
|---|---|---|
| ELM-8841 | MM Accountant Nurture (V1) | IES journey awareness for accountants showing intent |
| ELM-9388 | ACCT Nurture - Quarterly Content Launch | Full-funnel content-driven nurture for IES-ready accounts |
| ELM-9389 | ACCT Nurture - Monthly Event/Webinar Newsletter | Event-driven FAQ nurture for webinar-sourced leads |
| ELM-9390 | ACCT Nurture - Monthly Evergreen "Moment in Time" | Milestone-timed IES content delivery |
| ELM-9391 | ACCT Nurture - Amplify Feature Demo | Visual demo content nurture |
| ELM-9392 | ACCT Nurture - Managed Firms | ABM + Relevvo-identified high-value targets |
| ELM-9394 | ACCT Nurture - Incentive Campaign | Discount/referral-driven conversion |
| ELM-9395 | ACCT Nurture - Re-engagement | Reactivation for cold/stalled pipeline leads |

---

## 9. Technical Suggestions & Recommendations

> These are recommendations from the TPM to increase technical rigor. Discuss with GTM Tech and Marketing Ops.

### 9.1 Data Architecture
- **Define a canonical Accountant data model** in a shared schema registry (e.g., Confluence or a data catalog). Include field-level definitions, valid values, and ownership (who can write to each field). This prevents drift between Marketo, SFDC, and any downstream analytics warehouse.
- **Add `enrichment_timestamp` and `enrichment_source`** fields to the Marketo Person object for auditability. When debugging scoring issues, knowing when and how a lead was enriched is critical.

### 9.2 Enrichment Pipeline
- **Specify the BPP API contract** as a versioned OpenAPI spec. Include request/response schemas, error codes, rate limits, and SLA guarantees. This is a hard dependency for Phase 1 — without a finalized contract, development cannot begin.
- **Implement a dead-letter queue (DLQ)** for failed enrichment events. Failed enrichments should be retried asynchronously and surfaced in a GTM Tech monitoring dashboard (e.g., Datadog or Grafana).

### 9.3 Scoring Model
- **Parameterize all scoring weights** in a configuration layer (Marketo program tokens or an external config store) rather than hardcoding in Smart Campaigns. This allows Marketing to tune weights without engineering changes.
- **Add a score audit log** — every score change event should be logged with timestamp, trigger event, points added/removed, and resulting total. This is essential for debugging false MQL triggers and tuning the model.
- **Define a model validation plan**: before go-live, backtest the scoring model against 1H historical data. Measure precision/recall of MQL threshold against actual IES contract conversions.

### 9.4 L2A Matching
- **Document the matching algorithm** formally — including the confidence scoring formula, tie-breaking rules, and fallback behavior. This avoids ambiguity during implementation and QA.
- **Track L2A match rate** as a KPI. Target: >90% auto-resolution rate. If the unmatched queue exceeds 10% of volume, trigger an automated alert for data quality review.

### 9.5 SFDC Sync & Ops
- **Define the SFDC field mapping table** explicitly (Marketo field → SFDC field, sync direction, sync frequency). Include this as an appendix to the BRD.
- **Implement sync monitoring**: track sync latency p50/p95/p99 and failure rate. Alert on SLA breaches.

### 9.6 Observability
- **Build a lead funnel dashboard** showing: leads ingested → enriched → scored → MQL'd → Sales accepted → Meeting → Opportunity → Closed. Break down by firm tier and source channel. This is the primary metric surface for measuring BRD success.
- **Define SLIs/SLOs** for the pipeline: enrichment latency, sync latency, scoring accuracy, MQL-to-meeting conversion rate.

### 9.7 Testing Strategy
- **Require a UAT phase** per delivery phase. Define test scenarios covering: happy path, enrichment failure, L2A no-match, score threshold edge cases, nurture entry/exit, and SFDC sync conflict resolution.
- **Provide sample test data** with known expected outcomes for QA validation.

---

## 10. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | What is the finalized BPP API contract (request/response schema, SLA)? | GTM Tech | Open |
| 2 | What are the exact business rules for classifying `accountant_persona_type`? | Marketing | Open |
| 3 | What is the firm tier assignment logic — is it purely from SFDC segment, or does BPP provide an independent classification? | Sales Tech / Marketing | Open |
| 4 | What is the MQL score threshold per firm tier? Are there differentiated thresholds for National vs. HVAM vs. Unmanaged? | Marketing | Open |
| 5 | What SFDC fields need to be created or modified to support this integration? Needs Sales Tech sign-off. | Sales Tech | Open |
| 6 | What is the SLA for Sales follow-up after MQL? (Needed to define post-MQL nurture re-entry timing.) | Sales Innovation | Open |
| 7 | Is there a preferred webinar platform connector (ON24, Zoom, etc.) or are we using webhooks? | GTM Tech | Open |

---

## 11. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| BPP enrichment API not finalized in time for Phase 1 | Medium | High | Escalate to GTM Tech lead; define a manual enrichment fallback for initial launch |
| L2A match rate below target (<90%) | Medium | Medium | Pre-populate Firm Account records in SFDC from BPP seed data; tune matching algorithm iteratively |
| Scoring model produces excessive false MQLs | Medium | High | Backtest against 1H data before launch; implement a 2-week "shadow mode" where MQLs are logged but not routed |
| SFDC sync latency exceeds SLA during peak webinar events | Low | Medium | Pre-scale Marketo-SFDC sync capacity; implement queue-based sync with burst handling |

---

*This BRD is the foundational prerequisite for BRD-ACCT-ABM-001 (ABM Platform Integration). Phase 3 of this BRD depends on ABM signal availability defined in the companion BRD.*
