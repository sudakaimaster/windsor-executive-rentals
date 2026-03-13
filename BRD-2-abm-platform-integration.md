# BRD-2: ABM Platform Integration & Signal Orchestration — Accountant BU

| Field | Value |
|---|---|
| **Document Owner** | *(TBD)* |
| **Last Updated** | *(TBD)* |
| **Release Status** | Draft |
| **Initiative ID** | BRD-ACCT-ABM-001 |
| **Upstream Dependency** | BRD-ACCT-MKT-001 (Marketo Accountant Lead Management — Phase 2) |

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

This BRD defines the technical requirements for integrating the Account-Based Marketing (ABM) platform into the lead management tech stack for the Accountant BU. ABM campaigns targeting HVAM firms are already in flight and exceeding CTR benchmarks — 41% of accountant firms (1.4k) are engaging with ABM ads. However, due to a lack of system integration, engagement signals are siloed: firms showing high intent are not being contacted during the critical "golden window," and ABM influence on pipeline cannot be measured.

**Current State (As-Is):**
- ABM POCs are running for HVAM accounts, but contact extraction and SFDC import are **fully manual** (CSV exports, manual import requests to Marketing Ops).
- ABM engagement data (impressions, clicks, account-level intent scores) is trapped in the ABM platform and not flowing into Marketo or SFDC.
- No closed-loop reporting: ABM spend cannot be attributed to meetings, opportunities, or revenue.
- SFDC account/opportunity data is not fed back into ABM for targeting refinement or suppression.
- Nurture activation from ABM account lists requires manual list pulls and Marketo program enrollment.

**Future State (To-Be):**
- ABM engagement signals (account-level and contact-level) flow into Marketo in near-real-time via API integration.
- ABM signals contribute to the accountant lead scoring model (defined in BRD-ACCT-MKT-001) as a first-class scoring dimension.
- Contact extraction from engaged firms is automated: when a firm engages, all known decision-maker contacts are pulled and enrolled in Marketo nurture.
- SFDC data (account stage, owner, opportunity status, custom attributes) is ingested by ABM for dynamic audience segmentation and suppression.
- Closed-loop attribution: ABM engagement is connected to pipeline and revenue outcomes, with "new meetings" as the primary MQL-proxy KPI.

**Estimated Addressable Reach:** 80,000 decision-makers within 3,200 firms (~25 decision-makers/firm), representing ~90,000 IES-ready clients collectively.

---

## 4. Scope

### 4.1 In Scope

| # | Capability | Description |
|---|---|---|
| 1 | ABM → Marketo Signal Ingestion | Ingest account-level and contact-level engagement signals from the ABM platform into Marketo via API (webhook or batch sync). |
| 2 | ABM → Marketo Contact Extraction | When an account engages (surpasses intent threshold), automatically extract all known decision-maker contacts from the ABM platform and create/update Marketo Person records. |
| 3 | ABM Scoring Integration | Add ABM engagement signals as a scoring dimension in the Marketo accountant scoring model. Define signal-to-score mappings. |
| 4 | SFDC → ABM Data Feed | Push SFDC account fields (stage, owner, segment, opportunity status, custom attributes) into ABM for audience selector refinement and suppression logic. |
| 5 | ABM → SFDC Opportunity Attribution | Ingest opportunity data into ABM to power multi-touch attribution and connect engagement signals to pipeline/revenue outcomes. |
| 6 | Automated Nurture Activation | When ABM identifies an engaged account, automatically enroll extracted contacts into the appropriate Marketo nurture stream without manual list pulls. |
| 7 | ABM Audience Segmentation via SFDC | Use SFDC fields (account stage, owner, custom attributes) as selectors in ABM audience definitions to improve target precision. |
| 8 | Closed-Loop Reporting | Build attribution model connecting ABM engagement → MQL → Meeting → Opportunity → Revenue. Primary KPI: incremental new meetings attributable to ABM. |

### 4.2 Out of Scope

| # | Item | Rationale |
|---|---|---|
| 1 | Marketo schema changes, enrichment pipeline, L2A logic, base scoring model | Covered in BRD-ACCT-MKT-001 |
| 2 | ABM campaign strategy, creative, or media buying | Marketing Ops responsibility |
| 3 | ABM platform procurement or vendor selection | Assumes current ABM vendor is retained |
| 4 | DSP ad operations (bid management, audience creation in DSP) | Media team responsibility |

---

## 5. Technical Requirements

### 5.1 ABM → Marketo Integration (Signal Ingestion)

| Requirement ID | Description | Details |
|---|---|---|
| ABM-ING-001 | Integration method | Implement a near-real-time integration using one of: (a) ABM platform webhook → Marketo REST API, (b) ABM platform native Marketo connector (if available), (c) middleware (e.g., Workato, Tray.io) polling ABM API on a configurable schedule (minimum: every 15 minutes). Document the chosen pattern and rationale. |
| ABM-ING-002 | Signal schema | Define the canonical signal payload schema. Minimum fields: `account_id` (ABM), `account_name`, `domain`, `signal_type` (enum: `impression`, `click`, `site_visit`, `intent_surge`, `engagement_score_change`), `signal_value` (numeric), `signal_timestamp` (ISO 8601), `contact_email` (nullable), `contact_name` (nullable). |
| ABM-ING-003 | Signal deduplication | Deduplicate signals using a composite key of `account_id + signal_type + signal_timestamp` (truncated to 1-hour window). Prevent duplicate score inflation from replayed or batched events. |
| ABM-ING-004 | Ingestion SLA | Signals must be available in Marketo within **30 minutes** of the engagement event in the ABM platform. |
| ABM-ING-005 | Error handling | Failed ingestion events must be written to a dead-letter queue (DLQ). Implement automated retry (3 attempts, exponential backoff). Alert GTM Tech on DLQ depth > 100 events via PagerDuty or Slack. |
| ABM-ING-006 | Volume estimation | Estimate peak signal volume: ~1.4k engaged firms × ~25 contacts/firm × ~5 signal types × daily frequency = ~175k signals/day at peak. Validate that Marketo API rate limits and webhook throughput can handle this. Document the Marketo REST API daily call limit and plan for capacity. |

### 5.2 Automated Contact Extraction

| Requirement ID | Description | Details |
|---|---|---|
| ABM-EXT-001 | Engagement threshold trigger | Define an account-level engagement score threshold (configurable) that triggers contact extraction. When an account's composite ABM engagement score crosses the threshold, initiate extraction. |
| ABM-EXT-002 | Contact extraction logic | Query ABM platform API for all known contacts within the engaged firm. Filter to decision-maker personas only (using ABM job title/seniority data). Create or update Marketo Person records for each extracted contact. |
| ABM-EXT-003 | Marketo Person creation | For each extracted contact, set: `is_accountant = true`, `lead_source = 'ABM'`, `abm_account_id`, `abm_engagement_score`. Trigger BPP enrichment (per BRD-ACCT-MKT-001 MKT-ENR-001) for persona classification and firm tier assignment. |
| ABM-EXT-004 | Duplicate handling | Before creating a Marketo Person, check for existing records by email (primary) and name + firm domain (secondary). If a match exists, update the existing record with ABM engagement data rather than creating a duplicate. |
| ABM-EXT-005 | Extraction rate limit | Implement rate limiting on contact extraction to avoid overwhelming Marketo API (max 300 API calls/minute per Marketo REST API limits). Queue extraction jobs and process sequentially with backpressure. |

### 5.3 ABM Scoring Dimension (Marketo Integration)

| Requirement ID | Description | Details |
|---|---|---|
| ABM-SCR-001 | ABM score component | Add an `abm_engagement_score` field to the Marketo Person object. This is a sub-score that feeds into the composite accountant lead score defined in BRD-ACCT-MKT-001 (MKT-SCR-001). |
| ABM-SCR-002 | Signal-to-score mapping | Define scoring weights for ABM signals: `impression` (+1), `click` (+5), `site_visit` (+8), `intent_surge` (+20), `engagement_score_change` (delta-based, configurable multiplier). These weights are **additive** to the Marketo behavioral score. |
| ABM-SCR-003 | Account-level to contact-level score propagation | When an account-level signal is received (no specific contact), distribute the score equally across all known contacts within that firm in Marketo. Document the propagation logic. |
| ABM-SCR-004 | Score cap | Cap ABM score contribution at **40 points** to prevent ABM engagement alone from triggering MQL without any direct Marketo engagement (email, webinar, form fill). This ensures Sales receives leads that have both ABM awareness AND direct engagement. |
| ABM-SCR-005 | Combined threshold | MQL triggers when: `marketo_behavioral_score + marketo_demographic_score + abm_engagement_score >= threshold` (threshold defined in BRD-ACCT-MKT-001 MKT-SCR-004). |

### 5.4 SFDC → ABM Data Feed

| Requirement ID | Description | Details |
|---|---|---|
| ABM-SFD-001 | SFDC fields to push to ABM | Push the following SFDC Account fields to ABM for audience segmentation: `account_stage`, `account_owner`, `account_segment` (National/HVAM/Unmanaged), `has_open_opportunity` (boolean), `opportunity_stage`, `last_meeting_date`, and any custom attributes defined by Sales Innovation. |
| ABM-SFD-002 | Sync method | Implement a scheduled batch sync (daily minimum) from SFDC → ABM via SFDC report export + ABM API import, or via middleware (Workato/Tray.io). Evaluate ABM platform's native SFDC connector for real-time sync. |
| ABM-SFD-003 | Suppression logic | Suppress accounts in ABM audience lists where: `opportunity_stage = 'Closed Won'` (within last 90 days), `opportunity_stage = 'Closed Lost'` (within last 30 days, configurable cool-down), or `account_owner` has flagged "Do Not Target" in SFDC. |
| ABM-SFD-004 | Dynamic audience refresh | ABM audiences must refresh within **24 hours** of SFDC data changes. New accounts meeting targeting criteria should be automatically added; accounts meeting suppression criteria should be automatically removed. |

### 5.5 ABM → SFDC Attribution (Opportunity Data)

| Requirement ID | Description | Details |
|---|---|---|
| ABM-ATT-001 | Opportunity data ingestion | Push SFDC Opportunity data into ABM: `opportunity_id`, `account_id`, `stage`, `amount`, `close_date`, `created_date`, `source`. This powers ABM's attribution reporting. |
| ABM-ATT-002 | Multi-touch attribution model | Implement a time-decay multi-touch attribution model: ABM engagement touches within 90 days prior to opportunity creation receive attribution credit, weighted by recency (most recent touch = highest weight). |
| ABM-ATT-003 | Attribution KPIs | Track and report: (a) influenced pipeline ($), (b) influenced revenue ($), (c) incremental new meetings attributable to ABM, (d) account engagement lift (engaged vs. non-engaged firm conversion rates), (e) cost per influenced opportunity. |
| ABM-ATT-004 | Attribution data store | Store attribution data in a queryable format (SFDC custom object or analytics warehouse) for Marketing and Sales leadership reporting. |

### 5.6 Automated Nurture Activation from ABM

| Requirement ID | Description | Details |
|---|---|---|
| ABM-NUR-001 | Account list auto-sync | ABM account lists (e.g., "HVAM Engaged Firms Q3") must automatically sync to Marketo as static or smart lists. When a firm is added to an ABM target list, all known contacts in Marketo for that firm are automatically eligible for nurture enrollment. |
| ABM-NUR-002 | Nurture trigger | When ABM contact extraction (ABM-EXT-002) creates new Marketo Persons, automatically enroll them in the four-touch introductory nurture sequence (as defined in current ABM POC workflow). |
| ABM-NUR-003 | Real-time engagement trigger | If an ABM signal of type `intent_surge` is received for an account, trigger an immediate high-priority nurture email to all contacts within that firm (bypass normal cadence throttling for this single touch). |
| ABM-NUR-004 | Nurture-to-MQL handoff | After the four-touch nurture sequence completes, evaluate composite score. If >= MQL threshold, pass to Sales immediately. If below threshold, transition to ongoing nurture stream (per BRD-ACCT-MKT-001 MKT-NUR-001). |

---

## 6. Use Case Detail: ABM Digital Ads Full-Funnel Flow

**Trigger:** DSP and LinkedIn ads target HVAM firms to reach decision-makers/buyer group across the firm.

**Processing Steps:**

1. **Ad Engagement** — A decision-maker within a target firm clicks a DSP or LinkedIn ad. The ABM platform captures the engagement signal (click, site visit, or impression).
2. **Intent Score Update** — ABM platform updates the account-level engagement score. If the score crosses the configurable engagement threshold, trigger contact extraction (ABM-EXT-001).
3. **Contact Extraction** — ABM platform API returns all known decision-maker contacts for the engaged firm. Each contact is created/updated in Marketo with `lead_source = 'ABM'` and `abm_engagement_score` populated (ABM-EXT-002, ABM-EXT-003).
4. **Enrichment & L2A** — BPP enrichment fires for each new contact (per BRD-ACCT-MKT-001). L2A resolves each contact to the Firm Account in SFDC.
5. **Scoring** — ABM engagement score is added to the Marketo composite score. Combined with any existing Marketo behavioral/demographic scores.
6. **Nurture Enrollment** — Extracted contacts are enrolled in the four-touch ABM nurture sequence.
7. **MQL Evaluation** — After nurture or on subsequent engagement, if composite score >= threshold, trigger MQL and lead pass to Sales.
8. **Attribution Capture** — All ABM engagement touches are logged. When an opportunity is created, ABM attribution model calculates influenced pipeline credit.
9. **Closed-Loop Feedback** — Opportunity outcome (won/lost) syncs back to ABM to refine targeting and suppression.

---

## 7. Use Case Detail: Nurture Activation & Email Measurement

**Objective:** Unify ABM engagement data with Marketo email/form-fill/engagement data to provide a single view of prospect activity and enable automated, data-driven nurture.

**Processing Steps:**

1. **Unified Engagement Profile** — Merge ABM engagement signals with Marketo activity log for each contact. Store a unified `engagement_history` view accessible in both systems.
2. **Auto-Save & Activate** — When ABM identifies a prospect via account list targeting, automatically save the prospect to Marketo (if not already present) and activate them for nurture eligibility.
3. **Real-Time Engagement Trigger** — If a prospect shows engagement in ABM (e.g., multiple site visits within 24 hours), trigger a real-time Marketo nurture email (bypassing standard cadence for a single priority touch).
4. **Email Performance by ABM Segment** — Track email open rate, click rate, and conversion rate segmented by ABM engagement level (high/medium/low/none). This measures whether ABM-sourced leads respond differently to nurture.
5. **Attribution Integration** — Email engagement (opens, clicks, replies) feeds back into ABM as an additional signal for account-level scoring refinement.

---

## 8. Integration Architecture (Recommended)

```
┌─────────────┐      Engagement Signals       ┌──────────────┐
│             │ ──────────────────────────────► │              │
│  ABM        │      (webhook / API poll)       │   Marketo    │
│  Platform   │                                 │              │
│             │ ◄────────────────────────────── │              │
│             │      Contact lists, nurture     │              │
│             │      engagement data            │              │
└──────┬──────┘                                 └──────┬───────┘
       │                                               │
       │  Account data,              Lead score,       │
       │  opportunity data           MQL events,       │
       │  (daily batch)              lifecycle stage    │
       │                             (near-real-time)   │
       ▼                                               ▼
┌──────────────────────────────────────────────────────────────┐
│                        Salesforce CRM                         │
│  Accounts · Contacts · Opportunities · Tasks · Custom Objects │
└──────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────┐
│  Analytics Warehouse  │
│  (Attribution, KPIs)  │
└──────────────────────┘
```

**Key Integration Patterns:**
- ABM → Marketo: Webhook (preferred for near-real-time) or middleware polling (fallback).
- ABM → SFDC: Native connector or middleware batch sync (daily).
- Marketo → SFDC: Existing native Marketo-SFDC sync (enhanced with new fields).
- SFDC → ABM: Middleware batch sync (daily) for account/opportunity data.

---

## 9. Phased Delivery Plan

| Phase | Capability | Target Date | Dependencies | Acceptance Criteria |
|---|---|---|---|---|
| **Phase 1** | ABM → Marketo signal ingestion + contact extraction automation | *(TBD)* | BRD-ACCT-MKT-001 Phase 2 complete; ABM API access and documentation | Signals flow into Marketo within 30-min SLA; contacts auto-created with correct field population |
| **Phase 2** | ABM scoring integration + SFDC ↔ ABM data feeds | *(TBD)* | Phase 1 complete | ABM scores contribute to composite lead score; SFDC data refreshes ABM audiences within 24 hours; suppression logic active |
| **Phase 3** | Automated nurture activation + closed-loop attribution | *(TBD)* | Phase 2 complete | Nurture auto-enrollment operational; attribution model producing influenced pipeline/revenue reports |

---

## 10. Technical Suggestions & Recommendations

> These are recommendations from the TPM to increase technical rigor. Discuss with GTM Tech and Marketing Ops.

### 10.1 Integration Architecture
- **Choose webhook over polling wherever possible.** Polling introduces latency (minimum = poll interval) and wastes API calls when no new data exists. Webhooks provide near-real-time delivery and are more efficient. If the ABM platform supports webhooks, this should be the primary integration method.
- **Implement an integration middleware layer** (Workato, Tray.io, or a lightweight Lambda/Cloud Function) rather than direct point-to-point integrations. This provides: (a) centralized error handling, (b) transformation/mapping logic in one place, (c) retry and DLQ management, (d) observability. Direct ABM → Marketo API calls are brittle and hard to monitor.
- **Version the signal schema** (ABM-ING-002). Use a schema registry or at minimum a versioned JSON schema. When the ABM platform adds new signal types, the integration layer should handle unknown signal types gracefully (log and skip, don't fail).

### 10.2 Contact Extraction
- **Define the decision-maker filter criteria explicitly.** "Decision-maker" is vague. Specify the exact job titles, seniority levels, or ABM persona tags that qualify a contact for extraction. Example: `seniority IN ('C-Suite', 'VP', 'Director', 'Partner') AND job_function IN ('Accounting', 'Tax', 'Advisory', 'Finance')`.
- **Implement a contact extraction audit trail.** Log every extraction event with: firm, number of contacts extracted, number created vs. updated in Marketo, and any duplicates detected. This is critical for data quality monitoring.
- **Set a per-firm contact cap** (e.g., max 50 contacts/firm) to prevent data quality issues from firms with unusually large contact databases in the ABM platform.

### 10.3 Scoring
- **The ABM score cap (ABM-SCR-004) is critical — do not remove it.** Without a cap, a firm with high ABM engagement but zero direct engagement could flood Sales with unqualified MQLs. The cap ensures that ABM is an amplifier, not a standalone trigger.
- **Consider a composite scoring formula** rather than simple addition. For example: `final_score = behavioral + demographic + min(abm_score, cap) + interaction_bonus` where `interaction_bonus` is awarded when a contact has BOTH ABM engagement AND Marketo engagement (e.g., +10 bonus). This rewards multi-channel engagement.

### 10.4 Attribution
- **Do not attempt attribution without a clearly defined attribution window and model.** The BRD should specify: (a) lookback window (recommended: 90 days), (b) model type (recommended: time-decay for ABM, since recency matters), (c) credit allocation rules for multi-touch. Without this, attribution data will be contested by stakeholders.
- **Store raw attribution events, not just aggregates.** Raw events enable re-analysis under different attribution models. Store in an analytics warehouse (Snowflake, BigQuery, or Databricks) — not just in SFDC custom objects which are hard to query at scale.
- **Define a control group methodology.** To prove ABM incrementality, designate a holdout set of comparable firms that receive no ABM treatment. Compare meeting/opportunity rates between treated and holdout groups. Without a control group, "influenced pipeline" is a correlation metric, not a causal one.

### 10.5 Data Privacy & Compliance
- **Document PII handling for ABM-extracted contacts.** Contact extraction pulls names, emails, and job titles from a third-party ABM platform. Ensure this complies with your organization's data governance policies and any applicable privacy regulations (CAN-SPAM, CCPA, GDPR if applicable). Include a privacy review step in Phase 1.

### 10.6 Observability & Monitoring
- **Build an integration health dashboard** showing: signals received/day, extraction events/day, sync failures, DLQ depth, p95 ingestion latency, and Marketo API usage (% of daily limit consumed). This dashboard should be the first thing GTM Tech checks during campaign launches.
- **Define runbook for common failure scenarios**: ABM API downtime, Marketo API rate limit exceeded, SFDC sync failure, DLQ overflow. Include escalation paths.

### 10.7 Vendor-Specific Considerations
- **Document the specific ABM platform API capabilities and limitations.** Different ABM vendors (6sense, Demandbase, Terminus, RollWorks) have vastly different API maturity. The implementation plan will differ significantly based on the vendor. Include the vendor name and API documentation links in the BRD appendix.
- **Validate API rate limits** against the volume estimates in ABM-ING-006. If the ABM API has restrictive rate limits, the batch polling approach may be necessary even if webhooks are preferred.

---

## 11. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | Which ABM platform is being used? What are its API capabilities (webhook support, rate limits, contact extraction API)? | GTM Tech | Open |
| 2 | What is the account-level engagement score threshold for triggering contact extraction? | Marketing | Open |
| 3 | What constitutes a "decision-maker" for contact extraction filtering? Define job titles/seniority. | Marketing / Sales Innovation | Open |
| 4 | What SFDC custom attributes should be pushed to ABM for audience segmentation? | Sales Tech | Open |
| 5 | Is there an existing middleware platform (Workato, Tray.io, etc.) in the GTM Tech stack, or does one need to be provisioned? | GTM Tech | Open |
| 6 | What is the attribution reporting destination — SFDC dashboards, Tableau, Looker, or another BI tool? | Marketing Analytics | Open |
| 7 | Has a privacy/data governance review been completed for ABM contact extraction? | Legal / Compliance | Open |
| 8 | What is the budget for ABM platform API usage (if usage-based pricing applies)? | Marketing | Open |
| 9 | Should ABM attribution data feed into Databricks for advanced analytics, or is SFDC/BI tool sufficient? | GTM Tech / Analytics | Open |

---

## 12. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| ABM platform API does not support webhook-based signal delivery | Medium | Medium | Fall back to middleware polling (15-min interval); accept higher latency |
| Contact extraction overwhelms Marketo API rate limits during campaign spikes | Medium | High | Implement backpressure queuing (ABM-EXT-005); pre-request Marketo API limit increase for campaign windows |
| ABM engagement data is too noisy (high impression volume, low signal quality) | Medium | Medium | Apply minimum signal quality filter (e.g., ignore impressions, only ingest clicks + site visits + intent surges for scoring) |
| Closed-loop attribution is contested by stakeholders without control group | High | Medium | Implement holdout group methodology (see 10.4); present incrementality analysis alongside influence metrics |
| SFDC → ABM daily sync is insufficient for suppression (recently closed deals still targeted) | Low | Medium | Evaluate real-time SFDC → ABM sync for critical suppression fields; or implement a 2x/day sync schedule |
| Data privacy concern with ABM-extracted contact PII | Low | High | Conduct privacy review in Phase 1; implement consent verification step if required |

---

## 13. Dependency Map

```
BRD-ACCT-MKT-001                        BRD-ACCT-ABM-001
(Marketo Integration)                    (ABM Platform Integration)
━━━━━━━━━━━━━━━━━━━━                     ━━━━━━━━━━━━━━━━━━━━━━━━

Phase 1: Schema + Enrichment ─────┐
                                  │
Phase 2: Scoring + L2A ───────────┼──►  Phase 1: Signal Ingestion +
                                  │              Contact Extraction
Phase 3: Nurture + IPD ───────────┘
                                        Phase 2: ABM Scoring +
                                                 SFDC ↔ ABM Feeds

                                        Phase 3: Nurture Activation +
                                                 Attribution
```

*BRD-ACCT-ABM-001 Phase 1 cannot begin until BRD-ACCT-MKT-001 Phase 2 is complete, because ABM-extracted contacts require the Marketo schema, enrichment pipeline, and L2A logic to be operational.*

---

*This BRD is a companion to BRD-ACCT-MKT-001 (Marketo Accountant Lead Management). Together they form the complete tech stack enablement for the Accountant BU 2H demand generation strategy.*
