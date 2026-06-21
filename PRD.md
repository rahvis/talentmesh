# TalentMesh — Product Requirements Document (Recruiter Experience)

> **A multi-agent, human-in-command recruiting platform.** This PRD specifies the
> recruiter-facing web application: what the recruiter sees, the end-to-end recruiter
> workflow, the screens (with low-fidelity ASCII wireframes), and everything a recruiter
> can do. It is grounded in the TalentMesh design blueprint and the synthetic demo data
> The data files ship a full synthetic library — 100 requisitions, 1,000 candidates, 33
> compliance rules, 17 interview rubrics, 17 locally-runnable A2A vendor agents, 508 comp
> bands, and 316 knowledge-base docs — and the §8 wireframes are built against a stable
> canonical subset of it (4 reqs, 8 candidates, 5 vendors, 8 rules) so the screen examples
> stay consistent. See §13 for the full grounding.

---

## Table of contents

1. Overview & document control
2. Problem & opportunity
3. Goals, non-goals & success metrics
4. Personas
5. Product principles
6. The recruiter's product surface (map of screens)
7. Recruiter end-to-end workflow (day in the life + funnel walkthrough)
8. Screen-by-screen functional spec + wireframes
   - 8.1 Global shell, login & workspace home
   - 8.2 Pipeline / Mission Control
   - 8.3 Intake & JD Builder
   - 8.4 Approvals Inbox
   - 8.5 Sourcing / Candidate Search
   - 8.6 Candidate 360
   - 8.7 Candidate Engagement / Echo monitor
   - 8.8 Compliance & Audit / Replay
   - 8.9 Analytics
   - 8.10 Command bar / copilot & notifications
9. Capability catalog — everything the recruiter can do
10. Roles & permissions
11. States, edge cases & empty/error states
12. Non-functional requirements
13. Data model & grounding
14. Acceptance criteria
15. Future extensions (post-v1.0)
16. Glossary & appendix

---

## 1. Overview & document control

TalentMesh is a multi-agent agentic recruiting platform that compresses the hiring funnel — intake, sourcing, screening, assessment, engagement, scheduling, background checks, and onboarding handoff — from weeks into days, while keeping a human recruiter in command of every consequential decision. A team of 11 specialized AI agents does the heavy, repetitive work visibly, in real time, and grounded in the company's own job ladders, candidate corpus, policies, and compliance rules; the recruiter steers from a single mission-control surface and personally approves or overrides every advancement, rejection, and offer. The core promise is speed without surrender of judgment or auditability: the throughput of full automation, delivered as a defensible, human-governed process that stands up to 2026 AI-hiring regulation.

| Field | Value |
|---|---|
| Product | TalentMesh — multi-agent agentic recruiting platform |
| Document | Product Requirements Document (PRD) |
| Version | 1.0 |
| Status | Draft for review |
| Author | Product / Solutions Architecture |
| Primary persona | Riya — Technical Recruiter |
| Last updated | 2026-06-21 |
| Related docs | TalentMesh_Design.md |

**Scope of this PRD.** This PRD focuses on the **recruiter-facing web application** — what the recruiter sees, does, and decides across the six recruiter screens (Pipeline / Mission Control, Intake & JD Builder, Approvals Inbox, Candidate 360, Compliance & Audit, Analytics) plus the candidate chat surface. The agent backend (the 11-agent LangGraph/ADK/A2A system), the separate candidate-facing app, and the compliance-officer views are referenced only where they intersect what the recruiter experiences; their full specification lives in `TalentMesh_Design.md`.

## 2. Problem & opportunity

Hiring is one of the highest-leverage and most administratively punishing things a company does, and the recruiter absorbs the punishment. The **average global time-to-hire is ~44 days**, and a recruiter spends an estimated **13 hours per week just sourcing** candidates for a *single* open role — before a single interview is scheduled. The **average cost per hire is ~$4,700**, and every extra week a role stays open compounds that loss through delayed productivity. Speed is not a nicety: **52% of candidates** say they will walk away from a slow, disjointed, or impersonal process, so administrative drag directly costs the best people. Volume roles make it worse — a single support req like REQ-2026-0478 draws **200+ applications per week** — and a legacy keyword-matching ATS routinely filters out strong candidates who simply used the "wrong" word. Meanwhile the recruiter context-switches across LinkedIn, ATS, calendar, and email all day, and a candidate like Wei Chen (0.63 vs a 0.70 threshold) can be auto-rejected by a blunt tool when the right answer is a thirty-second human look.

**2026 is the year AI hiring became a regulated activity**, which is simultaneously the opportunity and the constraint. The **EU AI Act** classifies recruitment/hiring/promotion AI as **high-risk**, with deployer obligations live from **2 August 2026**: mandatory **human oversight** (AI may assist but must not autonomously make the final advancement or rejection — `COMP-EUAI-02`), an auditable **reasoning log** for every ranking that a trained person can review and override (`COMP-EUAI-01`), and penalties up to **€35M or 7% of global turnover**. **NYC Local Law 144** requires an **annual independent bias audit** of selection rates and impact ratios across sex and race/ethnicity, public posting, and **candidate notice at least 10 business days** before an automated tool is used — with liability sitting on the **employer**, not the vendor (`COMP-LL144-01/02`). The **Illinois AI Video Interview Act** demands explicit candidate consent before any AI analysis of video (`COMP-ILVIA-01`), and **EEOC Title VII** disparate-impact doctrine requires every must-have criterion to be job-related and consistent with business necessity (`COMP-EEOC-01`). The core tension is sharp: recruiters need agentic speed to survive the volume, but **cannot ship an unauditable black box** that makes opaque, potentially biased decisions.

**Opportunity.** TalentMesh's wedge is to win on all four axes at once — **fast** (agents do sourcing, parsing, scoring, scheduling, and communication concurrently), **coordinated** (one orchestrator routing 11 specialists instead of the recruiter stitching tools by hand), **grounded** (every answer cited to the company's own data via RAG, not a model's guess), and **provably governed** (a human makes every final call, backed by an append-only reasoning log and time-travel replay). It is the only honest way to give a recruiter agentic throughput without asking them to deploy a system they cannot defend to legal — automation that is fast *because* it is legible, and deployable *because* the human stays in command.

## 3. Goals, non-goals & success metrics

**Goals**

- Give the recruiter **one mission-control surface** where every live pipeline, across all 12–18 of their open reqs, is visible and updates in real time as agents move candidates.
- Make every human-decision point a **one-click approve / override / request-changes** action in a single Approvals Inbox — JD approvals, shortlist approvals, borderline reviews, and offer approvals.
- Let agents do the busywork **visibly and legibly** — the recruiter can always see what each agent is doing and why, via a plain-language activity feed.
- Produce a **defensible audit trail** by default: an append-only reasoning log and replayable decision path for every consequential decision, with no extra recruiter effort.
- **Cut time-to-shortlist** dramatically by parallelizing sourcing and sequential screening so a ranked, rationale-backed shortlist arrives in days, not weeks.
- **Raise candidate response and completion rates** through fast, personalized, anti-ghosting engagement grounded in company knowledge.
- **Return hours to high-value work** by removing manual sourcing, resume parsing, scheduling, and status-chasing from the recruiter's day.
- Keep **fairness and compliance woven in**, not bolted on — bias scans, impact-ratio math, PII segmentation, and required candidate notices enforced inside the workflow.

**Non-goals (v1.0)**

- Not a payroll, benefits, or HRIS system of record — TalentMesh hands off to HRIS/IT vendors over A2A but does not replace them.
- Not the build-out of the candidate-facing app — the candidate chat surface is referenced only where it intersects the recruiter; its full spec is out of scope here.
- Not autonomous hiring — no agent may auto-reject or auto-hire; every advancement and rejection requires explicit human approval.
- Not a replacement for the recruiter's judgment — agents assist and recommend; the recruiter decides.
- No fine-tuned models — grounding is via RAG over company data for auditability and freshness, not retraining.
- Not a sourcing-data marketplace or external job-board aggregator in v1.0 (noted as a future A2A-server extension).

**Success metrics / KPIs**

| Metric | Definition | Target |
|---|---|---|
| Time-to-shortlist | Calendar time from req opened to a recruiter-approved ranked shortlist | ≤ 3 business days (from a ~44-day TTH baseline) |
| Recruiter sourcing hours per req | Manual hours the recruiter spends sourcing one role | ≤ 2 hrs/req (from ~13 hrs/week baseline) |
| Recruiter-hours-saved per req | Hours of busywork (sourcing, parsing, scheduling, status-chasing) returned per req | ≥ 10 hrs/req |
| Candidate response rate | % of contacted candidates who reply to outreach | ≥ 60% |
| % advancements with human approval | Stage advancements/rejections that passed through an explicit human approval | 100% (`COMP-EUAI-02`) |
| % decisions with complete reasoning log | Consequential decisions with a complete, replayable reasoning-log entry | 100% (`COMP-EUAI-01`) |
| Approval-queue latency | Median time an item waits in the Approvals Inbox before recruiter action | ≤ 4 business hours |
| Impact-ratio compliance | Reqs whose selection-rate impact ratios meet the bias-audit threshold (LL144) | 100% audited; ≥ 0.80 four-fifths ratio or flagged for remediation |
| Candidate-notice compliance | Candidates given the required ≥10-business-day AEDT notice + alternative-process option | 100% (`COMP-LL144-02`) |
| Time-to-fill | Calendar time from req opened to accepted offer | ≤ 21 days (from ~44-day baseline) |

## 4. Personas

| Persona | Role | Primary jobs-to-be-done | Touchpoints in the recruiter app |
|---|---|---|---|
| **Riya** *(primary)* | Technical Recruiter | Run all live pipelines; approve/override every decision; keep candidates warm; produce an audit trail | Pipeline / Mission Control, Approvals Inbox, Candidate 360, Intake & JD Builder, Analytics |
| **Marcus** *(secondary)* | Hiring Manager | Open a req in plain language; approve the AI-drafted JD; review a rationale-backed shortlist; approve offers | Intake & JD Builder; shortlist + offer items in the Approvals Inbox; read-only Candidate 360 |
| **Dana** *(oversight)* | HR / DEI Compliance Lead | Audit decisions; replay any candidate journey; monitor bias/impact ratios; export for LL144 | Compliance & Audit (reasoning-log viewer, replay, bias-audit dashboard) |
| **Sofia** *(candidate)* | Candidate (Senior Product Designer) | Get fast answers, fast scheduling, transparency, and never get ghosted | Candidate chat surface (Echo); appears as a card in Riya's pipeline and Candidate 360 |
| **Theo** *(builder)* | Platform Engineer | Keep the agent platform reliable, traceable, and safe to iterate | Not in the recruiter app — LangGraph Studio / LangSmith / ADK eval harness behind it |

**Riya — Technical Recruiter (primary user).**

*Context.* Riya owns **12–18 open reqs** at a time across engineering and design — for example REQ-2026-0431 (Senior Backend Engineer, L5, Austin, headcount 2, threshold 0.72), REQ-2026-0455 (Senior Product Designer, L5, Remote, threshold 0.70), and the clearance-gated REQ-2026-0490 (Staff Security Engineer, L6, DC, Secret, threshold 0.78). She juggles sourcing, screening, scheduling, and candidate experience simultaneously, and is now also accountable for *proving* her process is unbiased.

*Goals.* Keep every pipeline moving; surface the right candidates fast (an Amara Diallo at 0.91 or a Priya Nair at 0.86 should rise to the top automatically); make fast, confident advance/reject calls; never lose a strong candidate to a slow reply; and be able to hand legal a clean audit trail on demand.

*Pains.* Drowning in resumes (200+/week on a single volume req); constant context-switching across LinkedIn, ATS, calendar, and email; candidates going cold while she's heads-down elsewhere; legacy keyword ATS silently dropping good people; and the new, non-negotiable burden of documenting and defending every decision.

*What she needs from TalentMesh.* A **single mission-control screen** showing every pipeline live; agents that do the busywork **visibly** (so she trusts and can explain them); a **one-click Approvals Inbox** that puts the recommendation, the rationale, and the reasoning-log excerpt in front of her for each JD, shortlist, borderline case (e.g., Wei Chen at 0.63 routed to her, not auto-rejected), and offer; a **Candidate 360** with parsed resume, sub-scores with explanations, full interaction history, and check/compliance status; and an **audit trail she can hand to legal** without assembling it by hand.

*Success metric.* Time-to-shortlist, candidate response rate, and "hours returned to high-value work" — with 100% of her decisions backed by a complete, replayable reasoning log.

**Marcus — Hiring Manager (secondary).** An engineering lead who wants to open a req and then *not* think about it until great candidates appear. He kicks off intake in plain language ("I need two senior backend engineers for payments"), approves the AI-drafted JD in the Intake & JD Builder, and reviews a curated, rationale-backed shortlist plus offer approvals through the Approvals Inbox. His success metric is shortlist quality, low time spent, and calibrated interviews.

**Dana — HR / DEI Compliance Lead (oversight).** Accountable for surviving an EU AI Act conformity assessment and an LL144 bias audit. She lives in the **Compliance & Audit** view: the reasoning-log viewer (why each decision was made), the time-travel **replay** of any candidate's journey, and the **bias-audit dashboard** of selection rates and impact ratios from Lens, exportable for the annual audit. Her success metric is zero compliance gaps, defensible documentation, and demonstrable human oversight.

**Sofia — Candidate (one-liner).** A strong senior designer with options and no patience for bad process; she chats with the Echo assistant 24/7, gets fast scheduling, is never ghosted, and is clearly told a human reviews every decision.

**Theo — Platform Engineer (one-liner).** Maintains the agent platform itself using LangGraph Studio, LangSmith, and the ADK eval harness; he has no presence in the recruiter app but is why it stays reliable and traceable.

## 5. Product principles

- **Mission control, not a database.** The product is a live, steerable command surface where agents are visibly working — not a spreadsheet-style ATS the recruiter manually fills in.
- **AI assists, the human decides — always.** No agent auto-rejects or auto-hires; every advancement, rejection, and offer passes through a human approval, operationalizing the EU AI Act human-oversight mandate.
- **Every consequential decision is explainable and replayable.** An append-only reasoning log plus time-travel replay means any decision can be reconstructed and defended, by design rather than after the fact.
- **Agents are visible and legible.** The recruiter can always see, in plain language, what each agent is doing and why — trust comes from transparency, not from a hidden score.
- **Ground every answer in the company's own data.** Agents answer from RAG over the company's ladders, candidate corpus, policies, and compliance rules, with citations — current, attributable, and editable without retraining.
- **Fairness and compliance are woven in, not bolted on.** Bias scans on JDs and messages, impact-ratio math, PII segmentation, and required candidate notices live inside the workflow, not in a separate after-the-fact review.
- **Speed without ghosting.** Throughput never comes at the candidate's expense — fast, personalized engagement and an anti-ghosting nurture loop keep every candidate informed and respected.
- **Match the tool to the task.** The right model (and the right amount of automation) is applied to each step — cheap and fast where it's volume, careful and deliberate where the stakes are high — so speed never costs quality where quality matters.

## 6. The recruiter's product surface (map of screens)

TalentMesh is a **mission-control surface, not a spreadsheet ATS**. Where a legacy applicant-tracking system gives Riya a static grid of rows she has to refresh and manually push forward, TalentMesh gives her a *live* room: pipelines that move on their own as the 11 agents work, a streaming feed of what each agent is doing in plain language, and a single inbox of the handful of decisions only a human is allowed to make. The agents do the sourcing, parsing, scoring, scheduling, messaging, and checking; Riya steers and approves. Every consequential edge is wrapped by Aegis and logged, so the same surface that makes her fast also makes the process auditable.

### Information architecture / sitemap

```
TalentMesh (recruiter app)
|
+-- [Top nav]
|   +-- Pipeline / Mission Control ........ home (kanban + activity feed)
|   +-- Intake ............................ conversational JD builder
|   +-- Approvals ......................... HITL decision queue [badge]
|   +-- Sourcing .......................... Scout runs, channels, A/B yield
|   +-- Analytics ......................... funnel, source, hours saved
|   +-- Compliance ....................... reasoning log, replay, bias audit
|
+-- [Contextual / drill-in]
|   +-- Candidate 360 ..................... one candidate, everything
|   +-- Engagement / Echo monitor ......... live chats, nurture loops
|   +-- Settings .......................... reqs, team, roles, integrations
|
+-- [Global persistent UI]
    +-- Command / search bar  +  Notifications bell  +  Agent Activity rail
```

### Screen map

| Screen | Primary purpose | Powered by (agents) | Key real-time events |
|---|---|---|---|
| Pipeline / Mission Control | Live kanban of every req's funnel (Sourced -> Screened -> Assessment -> Interview -> Checks -> Offer); cards move as agents act | Atlas, Sift, Sync, Echo (feed surfaces all) | `stage.changed`, `agent.thinking`, `task.completed` |
| Intake & JD Builder | Describe a role in plain language; get a leveled, inclusive JD in a side-by-side editor; Approve & Open Req | Scribe (draft), Aegis (bias scan) | `agent.token`, `approval.required` |
| Approvals Inbox | Single queue of every human-decision point with recommendation + rationale + reasoning-log excerpt | Aegis (gate), Atlas (recommendation) | `approval.required`, `task.completed` |
| Sourcing | Watch Scout fan out across channels; compare A/B strategy yield; review surfaced candidates | Scout (Parallel), Atlas (routing) | `agent.thinking`, `task.completed`, `stage.changed` |
| Analytics | Funnel conversion, time-to-stage, source effectiveness, recruiter-hours-saved | Lens | `task.completed` |
| Compliance & Audit | Reasoning-log viewer, time-travel replay, impact-ratio bias dashboard (Dana's home) | Aegis, Lens | `task.completed`, `stage.changed` |
| Candidate 360 | One candidate: parsed resume, sub-scores w/ explanations, assessment, interaction history, check & schedule status, compliance flags | Sift, Gauge, Echo, Verify, Sync | `stage.changed`, `task.completed`, `agent.token` |
| Engagement / Echo monitor | Live candidate chats, outreach, and nurture-loop status; nudge a quiet candidate | Echo (Sonnet chat + Haiku notifications) | `agent.token`, `agent.thinking`, `task.completed` |
| Candidate chat surface (public) | 24/7 candidate Q&A (benefits, timeline, visa, "where am I?") with AI-assists/human-decides notice + LL144 alternative-process request | Echo | `agent.token`, `task.completed` |
| Settings | Manage reqs, team, role-based views (Recruiter / HM / Compliance / Admin), MCP/A2A integrations | Atlas (config) | `task.completed` |

### Global persistent UI

Across every screen sits a thin, always-on chrome layer. A **command / search bar** at the top doubles as Riya's natural-language copilot ("show me everyone stuck >5 days in screening"), routed through Atlas. A **notifications bell** collects A2A push events (a `CalendarCoordinator` booking confirmed, a `BackgroundCheckProvider` report completed) and surfaces approvals as toasts driven by `task.completed` and `approval.required` SSE events. The right-hand **Agent Activity Feed** rail streams what each agent is doing in plain language ("Sift scored 38 resumes for REQ-2026-0431", "Sync booked Amara's onsite") — LangGraph streaming surfaced as a human-readable feed. And an **approvals badge** rides the top nav, counting pending JD, shortlist, borderline, and offer decisions so Riya always knows how many calls are waiting on her.

## 7. Recruiter end-to-end workflow (day in the life + funnel walkthrough)

### 7.1 A day in Riya's life

It's 8:40 a.m. Riya opens **Pipeline / Mission Control**. The kanban is already in motion: overnight, Scout finished a parallel sourcing run on **REQ-2026-0478 (Customer Support Specialist, L2 Phoenix)** and the Agent Activity rail reads "Sift scored 38 resumes for REQ-2026-0478." The approvals badge in the top nav shows **3**.

She goes to the **Approvals Inbox** to clear them.

- **A JD draft from Scribe.** Marcus opened a new req in plain language; Scribe drafted the JD via RAG over the L5 ladder, and Aegis's inline banner reports the inclusive-language scan passed. Riya tweaks one responsibility line in the side-by-side editor and clicks **Approve & Open Req**, resolving the LangGraph `interrupt()`.
- **Wei Chen's borderline review.** Wei Chen (CAND-90120, applied to **REQ-2026-0455, Senior Product Designer**) scored **0.63 against the 0.70 threshold** — inside the borderline band, so the conditional edge routed her to a human instead of auto-rejecting. The card shows her sub-scores (strong Figma/prototyping, thin on design-systems and B2B SaaS) and the reasoning-log excerpt. Riya reads it, decides Wei is worth a recruiter screen given her trajectory, and clicks **Override -> advance**.
- **Amara's offer.** Amara Diallo (CAND-90105, **REQ-2026-0431**) is at **0.91**, sits in `values_interview`, and her background check already reads **passed**. The offer item aggregates her screen (0.91), assessment (0.88), and check status. Riya approves; Aegis finalizes the reasoning log.

With the inbox empty, she opens **Sourcing / the activity feed** and watches **Sift screen the 38 Phoenix resumes live** — parse and extract on Haiku, score on Sonnet — cards flowing from Sourced into Screened in real time. Daniel Osei (0.82) lands above the 0.65 CX threshold and advances; Jordan Lee (0.49) drops to the reject branch and enters Echo's 90-day nurture loop rather than getting ghosted.

Mid-morning she notices **Priya Nair** (0.86, REQ-2026-0431) has gone quiet since her technical assessment was scheduled. From the **Engagement / Echo monitor** she nudges her; Echo composes a contextual follow-up referencing Priya's payments-ledger work from long-term memory. Before lunch, a notification toast fires: the `CalendarCoordinator` has pushed back a booked multi-party onsite for Amara across 5 interviewers, and a `BackgroundCheckProvider` push flips **Helena Brandt's** check status in her Candidate 360. Riya never opened LinkedIn, a calendar, or her email client once.

### 7.2 The funnel, stage by stage — REQ-2026-0431 (Senior Backend Engineer, L5 Austin, threshold 0.72, headcount 2)

| # | Stage | Agent (model / ADK type) | What the agent does | What the recruiter sees | Recruiter action / approval |
|---|---|---|---|---|---|
| 1 | Intake | Scribe (Sonnet, LLM + artifact); Aegis bias scan | Turns Marcus's "I need two senior backend engineers for payments" into a leveled JD via RAG over the L5 ladder + `BAND-ENG-L5`; emits JD artifact; Aegis scans for exclusionary language | Conversational Intake panel; side-by-side JD editor; Aegis inclusive-language banner | **HITL: approve the JD** (resolves `interrupt()`) — *required* |
| 2 | JD approval | (Aegis -> Approvals Inbox) | Holds the JD at a consequential edge until a human signs off | Approvals Inbox item w/ rationale + reasoning-log excerpt | **HITL: Approve & Open Req** — *required* |
| 3 | Sourcing | Scout (Sonnet, ParallelAgent); Atlas routing | Fans out concurrently to internal `candidate_corpus`, LinkedIn (MCP), and referrals; de-dups; Atlas runs an A/B route of two strategies. Surfaces Amara (referral), Priya, Marcus Webb | Sourcing screen: live channel runs, A/B yield, surfaced candidate cards entering "Sourced" | Review channels/yield; optional override of strategy mix (no approval blocks the flow here) |
| 4 | Screening | Sift (Haiku parse/extract -> Sonnet score, SequentialAgent); LangGraph conditional edge; Aegis | `parse -> extract -> score -> rank`; conditional edge branches on score vs **0.72**: **Amara 0.91 advance**, a borderline (0.72-0.10 band) -> human, **Marcus Webb 0.58 -> reject path**. Aegis blocks any auto-reject and writes a per-candidate reasoning-log entry | Cards move Sourced -> Screened live; sub-scores + rationale on each card; Approvals Inbox gets shortlist + any borderline/reject confirmations | **HITL: approve the shortlist; confirm every borderline and every rejection** (no auto-reject) — *required* |
| 5 | Engagement | Echo (Sonnet chat + Haiku notifications, LLM + Loop) | Personalized outreach + confirmations; answers benefits/visa Qs via `company_knowledge` RAG with streaming; long-term memory keeps it contextual; quiet candidates enter the nurture LoopAgent | Echo monitor: live chats, nurture-loop status; Candidate 360 interaction history | Optional: nudge a quiet candidate (e.g., Priya); approve escalation to a video call |
| 6 | Assessment | Gauge (Opus, LoopAgent) | Issues a system-design assessment, scores it against **RUB-ENG-SYSDESIGN** (decomposition .25 / scalability .30 / tradeoffs .25 / communication .20; pass 0.70), iterating until the rubric is satisfied; optional bidirectional-streaming live coding | Candidate 360 assessment panel: rubric sub-scores + rationale (Amara 0.88) | Review results; advance decision flows through Approvals — *judgment* |
| 7 | Scheduling | Sync (Haiku, A2A client) -> `CalendarCoordinator` | Opens an A2A task (`find_mutual_slots` -> `book_interview`) for the 5-interviewer onsite; verifies the signed card; awaits push on completion | Activity feed "Sync booked Amara's onsite"; notification toast; Candidate 360 schedule status flips | None required (logistics); reschedule on request |
| 8 | Checks | Verify (Haiku orchestrate + Sonnet summarize, ParallelAgent) -> `BackgroundCheckProvider` + `ReferenceCheckAgent` | With consent, runs background + reference checks **concurrently over A2A** (signed cards; reference vendor negotiated to v0.3; long-running w/ push); Sonnet summarizes returned report artifacts | Candidate 360 check status (Amara: **passed**); `task.completed` toast on each report | **HITL: confirm consent**; review summarized reports — *required before offer* |
| 9 | Offer approval | Atlas aggregates state; Aegis | Aggregates screen (0.91), assessment (0.88), and passed checks into state; Aegis ensures no auto-hire and finalizes reasoning log | Approvals Inbox offer item with full rationale | **HITL: Riya/Marcus approve the offer** (no auto-hire) — *required* |
| 10 | Onboarding handoff | Bridge (Haiku, A2A client, Parallel) -> `HRISOnboarding` + `ITProvisioning` | On accept, fires the cross-vendor A2A handoff in parallel: `create_employee` / `trigger_onboarding` + `provision_by_role` (laptop/access); pushes status back | Candidate 360 / feed: onboarding + provisioning progress; `task.completed` toasts | None required; monitor handoff |

### 7.3 The human-decision boundary

By design, **no agent may auto-reject or auto-hire** (COMP-EUAI-02). The agents assist; a trained human makes every consequential call. Each of these points is a LangGraph `interrupt()` surfaced in the Approvals Inbox with the agent's recommendation, its rationale, and the reasoning-log excerpt:

1. **JD approval** — the JD is held at intake until Riya approves it (Aegis must pass its inclusive-language scan first, COMP-BIAS-01).
2. **Shortlist approval** — Riya signs off the ranked shortlist before anyone is advanced.
3. **Borderline review** — scores inside the threshold-minus-0.10 band (e.g., Wei Chen 0.63 vs 0.70) route to a human instead of being rejected; Riya advances or declines.
4. **Every rejection** — no candidate is auto-rejected; Riya confirms (Marcus Webb 0.58, Jordan Lee 0.49).
5. **Offer approval** — Riya/Marcus approve every offer; nothing is auto-hired.

**Why:** the EU AI Act classifies hiring AI as high-risk and mandates genuine human oversight (COMP-EUAI-02) plus a reviewable, overridable **reasoning log** for every ranking (COMP-EUAI-01). The append-only `reasoning_log` and time-travel replay make each of these decisions reconstructible for a conformity assessment.

### Swimlane sketch — Recruiter | Agents | External A2A

```
RECRUITER (Riya)        AGENTS (Atlas + team)        EXTERNAL A2A
-----------------       ----------------------       ---------------
 "2 backend eng" ----->  Scribe drafts JD
                         Aegis bias-scan
 approve JD      <-----  (interrupt: JD)
        |       ------>  Scout source (parallel)
                         Sift screen vs 0.72
 approve shortlist <---  (interrupt: shortlist)
 confirm reject   <---   Marcus W 0.58 -> reject
 borderline call  <---   (interrupt: human review)
                         Echo engage (stream)
                         Gauge assess vs RUB-ENG
                         Sync schedule  --------->  CalendarCoordinator
                                        <--------- push: booked
 confirm consent ----->  Verify checks  --------->  BG + Reference (v0.3)
                                        <--------- push: reports
 approve offer    <---  (interrupt: offer)
                         Bridge handoff --------->  HRIS + IT (parallel)
                                        <--------- push: provisioned
```

## 8. Screen-by-screen functional spec + wireframes

Each screen below is documented with a purpose, an ASCII wireframe populated with real requisition, candidate, comp, and compliance data from the source files, and a Notes block covering components, recruiter actions, and the live SSE events that update it. The wireframes are intentionally low-fidelity but layout-accurate: box positions, zones, and data placement reflect the shipping design. The primary user throughout is **Riya R.**, a Technical Recruiter; secondary roles (Marcus, Hiring Manager; Dana, HR/DEI Compliance; Theo, Platform Engineer) appear where they share a surface. All screens share a single app shell — wordmark, global command field, notifications, and a six-tab nav — so the product reads as one mission-control room rather than a database.

### 8.1 Global shell, login & workspace home

**Screen 8.1.1 — Sign-in (OAuth2 / OIDC, role selection)**

```
╔════════════════════════════════════════════════════════════════════════╗
║                          T a l e n t M e s h                           ║
║              agentic recruiting · human-in-command hiring               ║
╚════════════════════════════════════════════════════════════════════════╝
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│                         Sign in to your workspace                      │
│                                                                        │
│      ┌──────────────────────────────────────────────────────────┐     │
│      │  ▷  Continue with Google Workspace (OIDC)                 │     │
│      └──────────────────────────────────────────────────────────┘     │
│      ┌──────────────────────────────────────────────────────────┐     │
│      │  ▷  Continue with Okta (SAML / OIDC)                      │     │
│      └──────────────────────────────────────────────────────────┘     │
│      ┌──────────────────────────────────────────────────────────┐     │
│      │  ▷  Continue with Microsoft Entra ID                      │     │
│      └──────────────────────────────────────────────────────────┘     │
│                                                                        │
│      Email  [ riya.r@talentmesh.example                        ]       │
│                                                                        │
│      Sign in as ──────────────────────────────────────────────        │
│         ( ● ) Recruiter      ( ○ ) Hiring Manager                      │
│         ( ○ ) Compliance     ( ○ ) Admin                               │
│         ↳ role sets view scope + RBAC (Auth.js); Compliance            │
│           unlocks audit/replay; PII gate applies to all roles          │
│                                                                        │
│      ┌──────────────────────────────┐                                  │
│      │        Continue  ▷           │   Trouble signing in?            │
│      └──────────────────────────────┘                                  │
│                                                                        │
│   ⚠ Auth via OAuth2/OIDC (Auth.js). MFA enforced. By continuing you     │
│     accept the AI-assisted hiring notice (COMP-LL144-02 candidate       │
│     notice applies to applicants, not internal users).                 │
└────────────────────────────────────────────────────────────────────────┘
```

**Notes**
- **Purpose:** Authenticate internal users and bind them to a role that scopes the entire app via RBAC; gateway before any candidate PII is rendered.
- **Key components/zones:** Double-line product banner; federated IdP buttons (Google Workspace, Okta, Entra) for OAuth2/OIDC; email field; four-way role radio (Recruiter / Hiring Manager / Compliance / Admin) with scope explainer; Continue CTA; compliance footnote.
- **Recruiter can DO:** Choose an IdP, confirm role, sign in. Role selection determines landing view (Recruiter → workspace home; Compliance → Compliance & Audit).
- **Real-time / SSE:** None pre-auth. On success the client opens the authenticated SSE channel that powers every later screen.
- **Agents / back-end:** No agent runs here. Auth.js (NextAuth) handles OIDC token exchange and issues the session; role claim is read on every API call. The PII segmentation gate (COMP-PII-01) is enforced server-side regardless of role.

**Screen 8.1.2 — Workspace home (recruiter landing)**

```
╔════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                             🔔5   Riya R. ▾ ║
╠════════════════════════════════════════════════════════════════════════╣
║ [Pipeline] │ Intake │ Approvals │ Sourcing │ Analytics │ Compliance    ║
╚════════════════════════════════════════════════════════════════════════╝
┌─ Good morning, Riya ───────────────  Tue 21 Jun 2026 · 09:12 ──────────┐
│  My reqs: 2 · Team: +2    In pipeline: 6    ⚠ Approvals: 5             │
└────────────────────────────────────────────────────────────────────────┘
┌─ OPEN REQUISITIONS (◆mine ▸team) ──────────┬─ TODAY'S AGENDA (Sync) ───┐
│◆REQ-2026-0431 Sr Backend Eng L5 Austin     │ ● 10:30 Amara Diallo      │
│   ● HC 2 · P1 · thr 0.72 · 4 cand · ⚠1 appr│   onsite loop (5 intvw)   │
│◆REQ-2026-0455 Sr Product Designer L5 Remote │ ○ 13:00 Daniel Osei       │
│   ● HC 1 · P2 · thr 0.70 · 2 cand · ⚠1 appr│   HM interview (Phoenix)  │
│▸REQ-2026-0478 Cust Support Spec L2 Phoenix  │ ○ 15:30 Sofia Alvarez     │
│   ● HC 8 · P1 · thr 0.65 · 2 cand · 200+/wk │   design exercise debrief │
│▸REQ-2026-0490 Staff Security Eng L6 DC 🔒    │                           │
│   ● HC 1 · P1 · thr 0.78 · 1 cand · Secret  │ ▷ open full calendar      │
├─ QUICK KPIs ───────────────────────────────┴───────────────────────────┤
│ Time-to-screen ▓▓▓▓▓▓▓░░ 1.8d   Shortlist→offer ▓▓▓▓▓░░░ 9d            │
│ Pass-through  ▓▓▓▓▓▓░░░ 58%    LL144 impact ratio  ✓ within band       │
├─ AGENT ACTIVITY FEED (live) ───────────────────────────────────────────┤
│ 09:11 ● Sift   scored 38 resumes for REQ-0431 (Haiku→Sonnet)          │
│ 09:08 ✓ Sync   booked Amara Diallo onsite — 5 interviewers (A2A)      │
│ 09:04 ◐ Verify Helena Brandt background check in progress             │
│ 08:57 ⚠ Aegis  borderline: Wei Chen 0.63 < 0.70 → human review        │
│ 08:50 ★ Sift   recommends advancing Priya Nair 0.86 (REQ-0431)        │
│ ▷ open Pipeline / Mission Control                                      │
└────────────────────────────────────────────────────────────────────────┘
```

**Notes**
- **Purpose:** Orient Riya the moment she logs in — what is mine, what needs a decision today, what the agents have done since she left.
- **Key components/zones:** Standard app chrome (wordmark, command field, 🔔 5 notifications, Riya R. avatar, six-tab nav with Pipeline active); greeting strip with at-a-glance counts; Open Requisitions list — Riya (REC-08) owns REQ-2026-0431 and REQ-2026-0455 (◆); REQ-2026-0478 (REC-12) and REQ-2026-0490 (REC-03) appear via the shared team view (▸) — each with headcount, priority, threshold, candidate counts, and clearance flag on REQ-0490; Today's Agenda fed by Sync; Quick KPIs with block-meter bars; Agent Activity Feed preview.
- **Recruiter can DO:** Click into any req (→ Pipeline filtered), open the approvals badge (→ Approvals Inbox, 5 pending), open today's interviews, or jump to the full Agent Activity Feed.
- **Real-time / SSE:** `task.completed` and `agent.token` append rows to the activity feed; `approval.required` increments the ⚠ Approvals badge and 🔔 count; `stage.changed` updates per-req candidate counts; agenda items refresh on Sync booking events.
- **Agents / back-end:** Sift (screening), Sync (scheduling/agenda), Verify (checks), Aegis (borderline guardrail), Lens (KPI + LL144 impact ratio). Backed by the requisitions/candidates services and the LangGraph streaming channel surfaced as the feed.

### 8.2 Pipeline / Mission Control (home for the recruiter)

**Screen 8.2 — Pipeline / Mission Control (kanban + live agent feed)**

```
╔════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                             🔔5   Riya R. ▾ ║
╠════════════════════════════════════════════════════════════════════════╣
║ [Pipeline] │ Intake │ Approvals │ Sourcing │ Analytics │ Compliance    ║
╚════════════════════════════════════════════════════════════════════════╝
┌ Req: [REQ-2026-0431 Sr Backend Eng L5 ▾]  thr 0.72  HC 2  ⚠ Appr 1 ───┐
└────────────────────────────────────────────────────────────────────────┘
┌─ KANBAN ──────────────────────────────────────────┬─ AGENT FEED ◐live ─┐
│Sourced │Screen'd│Assess │Interv │Checks │ Offer    │ 09:11 ● Sift       │
│────────│────────│───────│───────│───────│──────────│  scored 38 resumes │
│┌──────┐│┌──────┐│┌─────┐│       │       │          │  for REQ-0431      │
││M.Webb││★Priya││      ││       │       │          │ 09:08 ✓ Sync       │
││ 0.58 ││ Nair ││      ││       │       │          │  booked Amara's    │
││✗ 0.72││ 0.86 ││      ││       │       │          │  onsite (5 intvw)  │
││below ││sys-dsn││      ││       │       │          │ 09:04 ◐ Verify     │
│└──────┘│└──────┘│      ││       │       │          │  Helena bg-check   │
│        │ 3d ▲   │      ││       │       │          │  in progress       │
│        │        │      ││       │       │ ┌──────┐ │ 08:57 ⚠ Aegis      │
│        │        │      ││       │       │ │      │ │  Wei Chen 0.63 <   │
│        │        │      ││       │       │ │      │ │  0.70 → review     │
│        │        │      ││       │       │ │      │ │ 08:50 ★ Sift       │
│        │        │      ││       │       │ │      │ │  rec advance Priya │
│        │        │      ││       │       │ └──────┘ │  Nair 0.86         │
└────────┴────────┴──────┴───────┴───────┴──────────┴────────────────────┘
┌─ ALL ACTIVE CANDIDATES (team / cross-req view) ───────────────────────┐
│ ★ Amara Diallo 0.91  REQ-0431  values_interview  bg ✓  ⚠ offer appr   │
│ ★ Helena Brandt 0.88 REQ-0490🔒 leadership_panel  bg ◐ in_progress     │
│   Priya Nair   0.86  REQ-0431  system_design      3d ▲ aging          │
│   Sofia Alvarez 0.84 REQ-0455  design_exercise    ─                   │
│   Daniel Osei  0.82  REQ-0478  hiring_manager      ─                   │
│ ⚠ Wei Chen     0.63  REQ-0455  screening BORDERLINE → human review    │
├─ PENDING APPROVALS (5) ────────────────────────────────────────────────┤
│ ⚠ Offer: Amara Diallo  · ⚠ Shortlist: REQ-0431  · ⚠ JD: REQ-0455       │
│ ⚠ Borderline: Wei Chen · ⚠ Advance: Priya Nair  →  [ Open Inbox ▷ ]    │
└────────────────────────────────────────────────────────────────────────┘
```

**Notes**
- **Purpose:** The hero screen — a single live board where Riya watches candidates move through the funnel and intervenes only at decision points. Looks like mission control, not a database.
- **Key components/zones:** Req filter bar (currently REQ-2026-0431, threshold 0.72, HC 2, 1 approval); six-column kanban (Sourced → Screened → Assessment → Interview → Checks → Offer) with candidate cards showing score, stage glyph, ✗ below-threshold marker (Marcus Webb 0.58 < 0.72), ★ AI-recommended (Priya Nair), and stage-aging "3d ▲"; right-rail Agent Activity Feed; a cross-req active-candidate roster (Amara 0.91, Helena 0.88, Priya 0.86, Sofia 0.84, Daniel 0.82, Wei Chen 0.63 flagged); pending-approvals strip with badge count 5.
- **Recruiter can DO:** Filter by req; drag a card forward only after approval (governance blocks silent moves); open a card → Candidate 360; click any approval chip → Approvals Inbox; click a feed line to inspect the originating agent run / reasoning log.
- **Real-time / SSE:** `stage.changed` moves cards between columns live; `agent.thinking` / `agent.token` stream the right-rail feed token-by-token; `task.completed` posts completion lines (e.g. Sync onsite booked); `approval.required` lights the pending-approvals strip and badge.
- **Agents / back-end:** Atlas orchestrates; Sift (screening scores), Gauge (assessment), Sync (interview coordination), Verify (checks), Aegis (borderline/guardrail), Lens (aging metrics). Powered by the LangGraph pipeline streaming over SSE; card positions come from each candidate's `stage`/`status` in the candidates service.

### 8.3 Intake & JD Builder

**Screen 8.3 — Intake & JD Builder (conversational left, JD editor right)**

```
╔════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                             🔔5   Riya R. ▾ ║
╠════════════════════════════════════════════════════════════════════════╣
║ Pipeline │ [Intake] │ Approvals │ Sourcing │ Analytics │ Compliance    ║
╚════════════════════════════════════════════════════════════════════════╝
┌─ SCRIBE · INTAKE CHAT ──────────────┬─ JD EDITOR · REQ-2026-0431 (draft)┐
│ Marcus (HM):                        │ Senior Backend Engineer · L5      │
│ "I need two senior backend          │ Payments Infrastructure · Austin  │
│  engineers for payments."           │ Hybrid (3 days) · HC 2 · P1       │
│                                     │───────────────────────────────────│
│ Scribe ◐ drafting…                  │ MISSION                           │
│ ↳ leveled to L5 (5–8 yrs, owns      │  Build and operate the payment    │
│   complex projects, leads design    │  rails that move money reliably   │
│   reviews) per leveling_guide       │  at 99.99% uptime.                │
│ ↳ pulled BAND-ENG-L5 comp band      │ THE TEAM                          │
│ ↳ RAG: past payments JDs +          │  Payments Infrastructure, Platform│
│   inclusive-language rules          │ WHAT YOU'LL DO                    │
│                                     │  • Design high-throughput payment │
│ You (Riya):                         │    processing services           │
│ "Keep it inclusive, widen the       │  • Own 99.99% reliability targets │
│  funnel — must vs nice-to-have."    │  • Mentor + lead design reviews   │
│                                     │ MUST-HAVES                        │
│ Scribe: separated must/nice,        │  Python · distributed systems ·   │
│ flagged 2 terms (see banner ▼).     │  PostgreSQL · Kafka · AWS · 5+ yrs│
│                                     │ NICE-TO-HAVES                     │
│ [ type a message…              ▷ ]  │  Go · payments · PCI-DSS · gRPC · │
├─ AEGIS · INCLUSIVE-LANGUAGE CHECK ──┤  Terraform                        │
│ ⚠ COMP-BIAS-01 — 2 issues found     │ COMP & BENEFITS                   │
│  ✗ "rockstar engineer"  → "skilled  │  Base $175,000–$215,000 + equity  │
│      engineer"        [Accept fix]  │  ~$90k/yr · 15% bonus target      │
│  ✗ "aggressive roadmap" → "ambitious│  (BAND-ENG-L5, tier_1)            │
│      roadmap"         [Accept fix]  │ EEO STATEMENT                     │
│  ✓ must/nice split widens funnel    │  Equal-opportunity employer; we   │
│  Scanned before recruiter review.   │  assess only job-related criteria.│
├─────────────────────────────────────┼───────────────────────────────────┤
│ ⛔ No agent may publish a JD without │ [ Accept all fixes ] [ Edit ]     │
│    human approval (COMP-EUAI-02).   │ [ ✦ Approve & Open Req ▷ ]        │
└─────────────────────────────────────┴───────────────────────────────────┘
```

**Notes**
- **Purpose:** Turn a manager's plain-language request ("I need two senior backend engineers for payments") into a structured, leveled, inclusive JD that Riya approves before the req opens.
- **Key components/zones:** Left conversational panel (Scribe drafting, showing its leveling + comp + RAG reasoning inline); right side-by-side JD editor for REQ-2026-0431 with the real template sections (mission, team, what you'll do, must-haves and nice-to-haves split per the JD template, comp band BAND-ENG-L5 $175k–$215k + ~$90k equity + 15% bonus, EEO statement); an Aegis inclusive-language banner flagging "rockstar" and "aggressive" with one-click fixes; governance footnote.
- **Recruiter can DO:** Chat to refine; accept individual or all inclusive-language fixes; edit any JD section; click **✦ Approve & Open Req** to publish and resolve the pending LangGraph interrupt that opens the requisition and releases it to Scout for sourcing.
- **Real-time / SSE:** `agent.thinking` shows Scribe's "drafting…" state; `agent.token` streams the JD into the editor as it is written; `approval.required` is the interrupt this screen exists to resolve; an Aegis pass emits the banner before the draft surfaces.
- **Agents / back-end:** Scribe (intake + JD, Sonnet 4.6) drafts as an ADK artifact via RAG over `job_knowledge` (leveling_guide, comp_bands, past JDs, inclusive rules); Aegis (Opus guardrail) enforces COMP-BIAS-01 and the COMP-EUAI-02 no-auto-publish rule; Atlas routes the approval back into the graph via `interrupt()`.

### 8.4 Approvals Inbox

**Screen 8.4 — Approvals Inbox (human-decision queue)**

```
╔════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                             🔔5   Riya R. ▾ ║
╠════════════════════════════════════════════════════════════════════════╣
║ Pipeline │ Intake │ [Approvals ⚠5] │ Sourcing │ Analytics │ Compliance ║
╚════════════════════════════════════════════════════════════════════════╝
┌─ PENDING (5) ──────────────────┬─ DETAIL · Offer approval ──────────────┐
│ ⚠ OFFER                        │ Amara Diallo · CAND-90105              │
│   Amara Diallo  REQ-0431  ◉    │ REQ-2026-0431 Sr Backend Eng L5        │
│   score 0.91 · bg ✓            │────────────────────────────────────────│
│ ────────────────────────────── │ ★ AGENT RECOMMENDATION (Atlas/Gauge)   │
│ ⚠ SHORTLIST                    │   Extend offer. Top of pipeline; all   │
│   REQ-0431 top candidates      │   gates cleared.                       │
│   Priya 0.86 / Amara 0.91      │                                        │
│ ────────────────────────────── │ RATIONALE                              │
│ ⚠ BORDERLINE REVIEW            │  • screen 0.91 ≥ thr 0.72  ✓           │
│   Wei Chen 0.63 vs 0.70        │  • assessment 0.88 (system design)  ✓  │
│   REQ-0455                     │  • background check  ✓ passed          │
│ ────────────────────────────── │  • values_interview  ✓ completed       │
│ ⚠ JD APPROVAL                  │ SUB-SCORES  ▓▓▓▓▓▓▓▓▓░ system-design .90│
│   REQ-0455 Sr Designer L5      │             ▓▓▓▓▓▓▓▓░░ mentorship   .85│
│ ────────────────────────────── │             ▓▓▓▓▓▓▓▓▓░ values fit   .89│
│ ⚠ ADVANCE                      │ REASONING-LOG EXCERPT (append-only)    │
│   Priya Nair 0.86 → HM         │  16:58 Gauge: rubric SYS-DESIGN 0.90   │
│   REQ-0431                     │  17:00 Verify: bg PASS (A2A signed)    │
│                                │  17:02 Aegis: PII gate clear; oversight│
│  Filter: [ All types ▾ ]       │         required → interrupt() raised  │
├────────────────────────────────┤  ▷ replay full decision (time-travel)  │
│ ⛔ No auto-hire / no auto-reject│────────────────────────────────────────│
│   EU AI Act human oversight  │ [ ✓ Approve ] [ ⨯ Override ] [ ↺ Changes ]│
│   COMP-EUAI-02 (blocking)      │ Decision is logged to your name (RBAC).│
└────────────────────────────────┴────────────────────────────────────────┘
```

**Notes**
- **Purpose:** The single queue where every consequential decision is made by a human — operationalizing the "AI assists, human decides" boundary across mixed approval types.
- **Key components/zones:** Left pending list (5 items of mixed type: Offer—Amara Diallo, Shortlist—REQ-0431, Borderline—Wei Chen 0.63 vs 0.70 threshold, JD—REQ-0455, Advance—Priya Nair) with a type filter; right detail pane showing the agent recommendation, rationale checklist against the real threshold/scores, sub-score meters, an append-only reasoning-log excerpt with a time-travel replay control, and the decision buttons; persistent ⛔ no-auto-hire/no-auto-reject banner.
- **Recruiter can DO:** Select any item; read the rationale and reasoning log; **Approve**, **Override** (e.g. advance Wei Chen despite the sub-threshold score, or decline a recommended offer), or **Request changes**; replay the full decision for audit. Every action is attributed to Riya via RBAC.
- **Real-time / SSE:** `approval.required` pushes new items into the queue and drives the ⚠5 badge / 🔔 count; resolving an item calls back into the graph and emits `stage.changed` on the Pipeline board; `agent.token` streams any freshly generated rationale.
- **Agents / back-end:** Atlas plus the recommending agent (Gauge for assessment/offer, Sift for shortlist/advance, Scribe for JD, Aegis for the borderline guardrail and PII gate). Each pending item corresponds to a LangGraph `interrupt()`; the recruiter's decision resumes the paused run, satisfying COMP-EUAI-01 (reasoning log) and COMP-EUAI-02 (human oversight).

### 8.5 Sourcing / Candidate Search (Scout)

One-line purpose: Semantic + metadata search workspace where Riya runs Scout's parallel sourcing across channels, compares A/B strategies, and adds surfaced talent to a pipeline.

```
╔══════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                               🔔5   Riya R. ▾ ║
╠══════════════════════════════════════════════════════════════════════════╣
║ Pipeline │ Intake │ Approvals │ [Sourcing] │ Analytics │ Compliance      ║
╚══════════════════════════════════════════════════════════════════════════╝
┌── Scout · Sourcing for REQ-2026-0490 Staff Security Eng L6 (DC) ─────────┐
│ ⌕ application security threat modeling FedRAMP python            [Search]│
│ Filters: ◆clearance=Secret  ▸location=DC  ▸level=L6  ▸source=any  +add   │
└──────────────────────────────────────────────────────────────────────────┘
┌── Ranked results (Scout · Sonnet · ParallelAgent fan-out) ───────────────┐
│  Match  Candidate          Source            Clearance  Loc      Action  │
│  ★0.92  Helena Brandt      internal_database  Secret ✓  DC      [+ pipe] │
│        9y · Sr AppSec Eng · GovCloud Systems · FedRAMP, threat modeling  │
│   0.74  R. Okafor          linkedin           Secret ✓  DC(hyb) [+ pipe] │
│   0.69  T. Nguyen          referral           none      Remote  [+ pipe] │
│   0.61  M. Santos          job_board          none      Austin  [+ pipe] │
│   ⚠ dup  H. Brandt (li)   linkedin   → merged into internal_database rec │
└──────────────────────────────────────────────────────────────────────────┘
┌── A/B sourcing strategy ─────────────┐┌── Channel effectiveness ────────┐
│ Strat A: clearance-filter-first      ││ internal_db ████████░░  4 hits  │
│   yield 4 qualified / 18 surfaced    ││ linkedin    █████░░░░░  3 hits  │
│ Strat B: skills-semantic-first       ││ referral    ███░░░░░░░  1 hit   │
│   yield 2 qualified / 27 surfaced    ││ job_board   ██░░░░░░░░  1 hit   │
│ ► A wins (22% vs 7%) → Atlas routes  ││ pass→pipe rate: A 22% B 7%      │
└──────────────────────────────────────┘└─────────────────────────────────┘
```

Notes:
- Purpose: Run Scout-powered talent discovery for an open req (here REQ-2026-0490, threshold 0.78, Secret-clearance required) and decide who enters the pipeline; clearance role forces the candidate_corpus retrieval to filter on clearance metadata.
- Key components/zones: semantic query box; metadata filter chips (active `clearance=Secret`); ranked results list with match score, source channel and clearance glyph; A/B strategy-comparison panel; channel-effectiveness mini bar chart; dedupe row.
- Recruiter can DO: edit the query, toggle/add filter chips, add a candidate to pipeline ([+ pipe]), merge/dedupe duplicate records, open a profile, and accept the winning A/B strategy that feeds Atlas routing.
- Real-time / SSE: `agent.thinking` (Scout fanning out across channels), `agent.token` (live result streaming), `task.completed` (each parallel channel returns), score badges and yield bars update as results stream in.
- Agents / back-end: Scout (Sonnet, ParallelAgent) runs the channel fan-out and ranking against the candidate_corpus; Atlas (Opus) ingests A/B yield to bias future routing; Aegis (Opus guardrail) holds back any auto-reject — surfacing only, no decisions here.

### 8.6 Candidate 360

One-line purpose: The single-candidate deep view for Amara Diallo — scores, evidence, assessment-vs-rubric, full history, checks and compliance — all on one screen.

```
╔══════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                               🔔5   Riya R. ▾ ║
╠══════════════════════════════════════════════════════════════════════════╣
║ [Pipeline] │ Intake │ Approvals │ Sourcing │ Analytics │ Compliance      ║
╚══════════════════════════════════════════════════════════════════════════╝
┌── Amara Diallo · CAND-90105 · REQ-2026-0431 Sr Backend Eng L5 ───────────┐
│ Sr Backend Engineer @CloudNimbus · 8y · Austin TX · src referral(EMP-204)│
│ Stage ● values_interview  bg ✓ passed   ◆ Recommendation: AUTO-ADVANCE   │
│ [ Message via Echo ]  [ Nudge stage ]  [ Open reasoning log ]            │
├──────────────────────────────────────────────────────────────────────────┤
│ Scores                          │ Resume & extracted skills              │
│ Screen      ████████▉ 0.91      │ Go · Python · PostgreSQL · Kafka · AWS │
│ Assessment  ████████▌ 0.88      │ distributed systems · PCI-DSS          │
│  thr 0.72 ✓ (both well above)   │ "PCI-DSS tokenization svc in Go,       │
│                                 │  30k req/sec; cut infra cost 28%;      │
│ Assessment vs RUB-ENG-SYSDESIGN │  led 5-eng team."                      │
│  decomposition  ████████ 4.4/5  ├────────────────────────────────────────┤
│  scalability    █████████ 4.6/5 │ Interaction history                    │
│  tradeoffs      ████████ 4.2/5  │ 05-30 ✉ Echo  referral fast-track warm │
│  communication  ████████ 4.3/5  │        intro re: PCI tokenization      │
│  weighted 0.88 ≥ 0.70 pass ✓    │ 06-15 ⧗ Sync  5-interviewer onsite     │
├─────────────────────────────────┤        loop booked via Calendar (A2A)  │
│ Scheduling  ✓ onsite loop booked│ Compliance: COMP-EUAI-02 human appr ☐ │
│ Background   ✓ passed (Verify)  │ COMP-PII-01 demo data segmented ✓     │
└─────────────────────────────────┴────────────────────────────────────────┘
```

Notes:
- Purpose: Give Riya everything needed to make a defensible human decision on one candidate — the highest scorer (0.91 screen / 0.88 assessment), already past background check and sitting at values_interview.
- Key components/zones: identity header with stage + recommendation; score meters vs the 0.72 req threshold; resume + extracted skills; assessment broken into RUB-ENG-SYSDESIGN dimensions (decomposition/scalability/tradeoffs/communication) vs the 0.70 pass bar; interaction-history timeline; scheduling + background panels; compliance flags with reasoning-log link.
- Recruiter can DO: message the candidate via Echo, nudge/advance the stage (which raises a human-approval gate), and open the append-only reasoning log; approve the pending COMP-EUAI-02 advancement checkbox.
- Real-time / SSE: `stage.changed` (loop progress), `task.completed` (Verify bg result, Sync booking), `approval.required` (advancement gate), score panels refresh when Gauge re-scores.
- Agents / back-end: Sift (Haiku→Sonnet) produced the screen score, Gauge (Opus, LoopAgent) the assessment vs rubric, Echo the messages, Sync the onsite booking, Verify the passed bg check, Aegis enforces PII segmentation and the human-approval gate; data assembled by Atlas.

### 8.7 Candidate Engagement / Echo Monitor

One-line purpose: Riya's two-pane window into Echo's candidate conversations — watch live RAG-grounded replies, ghost-risk and nurture status, and take over when needed.

```
╔══════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                               🔔5   Riya R. ▾ ║
╠══════════════════════════════════════════════════════════════════════════╣
║ [Pipeline] │ Intake │ Approvals │ Sourcing │ Analytics │ Compliance      ║
╚══════════════════════════════════════════════════════════════════════════╝
┌── Echo · conversations ──────┬── Sofia Alvarez · CAND-90051 · REQ-0455 ──┐
│ ●Sofia Alvarez   💬 web      │ ⚠ AI-assist notice shown (KB-FAQ-004)     │
│  "is this role remote?" 2m   │ ─────────────────────────────────────────│
│ ○Jordan Lee      ✉ email     │ Sofia 💬: Is the Sr Designer role fully   │
│  nurture · re-engage 90d ⚠   │   remote? I'm in Miami.                   │
│ ●Daniel Osei     📱 SMS      │ Echo: Yes — REQ-2026-0455 is fully remote │
│  "interview time?" 1h        │   (US). We add a $1,000 home-office       │
│ ●Priya Nair      ✉ email     │   stipend + co-working. [cite KB-CUL-001] │
│  hybrid + equity Q  3h ✓     │ Sofia 💬: Great. Can we talk team setup?  │
│ ────────────────────────────  │ Echo▌ Proposing a 15-min video chat…     │
│ Ghost-risk: Daniel ⚠ (1h no  │   ⮕ UX-negotiation: escalate to video?    │
│  reply to SMS reminder)      │   [ Approve outreach ] [ Escalate video ] │
│                              │ ─────────────────────────────────────────│
│ [ Trigger nurture loop ]     │ [ Take over ] [ Send on behalf ] [ Nudge ]│
└──────────────────────────────┴───────────────────────────────────────────┘
```

Notes:
- Purpose: Supervise Echo's autonomous candidate engagement across web/SMS/email, intervene on high-stakes moments, and keep quiet candidates warm — without breaching the AI-assist disclosure rule.
- Key components/zones: left conversation list with channel glyphs, last message, ghost-risk and nurture flags; right open streaming thread (Sofia asking about remote work, Echo answering from company_knowledge RAG with a `KB-CUL-001` citation); UX-negotiation escalation prompt; mandatory AI-assist notice banner (`KB-FAQ-004`); nurture status for Jordan Lee (re-engage in 90 days).
- Recruiter can DO: take over / send on behalf, approve outreach copy, escalate to a 15-min video call, trigger the nurture loop, and nudge a thread.
- Real-time / SSE: `agent.token` (Echo's streaming reply), `agent.thinking` (RAG retrieval), `task.completed` (message sent / video booked), ghost-risk and nurture badges update live.
- Agents / back-end: Echo (Sonnet for drafting + Haiku for quick acks) generates replies grounded in the company_knowledge RAG; Sync (Haiku) books any escalated video; Aegis (Opus) enforces COMP-LL144-02 candidate notice and COMP-BIAS-01 message scanning before send; LoopAgent drives the 90-day nurture re-engagement.

### 8.8 Compliance & Audit / Replay

One-line purpose: Dana's governance home (read access for Riya) — reasoning-log viewer, LangGraph time-travel replay, and the four-fifths-rule bias dashboard with notice and PII status.

```
╔══════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                               🔔5   Dana K. ▾ ║
╠══════════════════════════════════════════════════════════════════════════╣
║ Pipeline │ Intake │ Approvals │ Sourcing │ Analytics │ [Compliance]      ║
╚══════════════════════════════════════════════════════════════════════════╝
┌── Reasoning log · Wei Chen CAND-90120 · REQ-0455 (COMP-EUAI-01) ─────────┐
│ 06-12 16:00 Sift/Haiku  screen=0.63 vs thr 0.70 → BORDERLINE            │
│ 06-12 16:00 Atlas/Opus  no auto-reject (COMP-EUAI-02) → route to human  │
│ 06-12 16:00 Aegis/Opus  PII segmented (COMP-PII-01) ✓ ; notice sent ✓   │
│ append-only · model+ts stamped · override available to trained reviewer  │
├──────────────────────────────────────────────────────────────────────────┤
│ Replay / time-travel (LangGraph checkpoints)                             │
│  ►│ ├──●──────●──────●──────◐──────○──────○──┤  step ◀ ▶                 │
│     intake  screen  assess  HITL   sched   offer                         │
│  state@checkpoint-3 (assess): score 0.88, stage values_interview         │
├──────────────────────────────────────────────────────────────────────────┤
│ Bias audit · selection rates + impact ratio (COMP-LL144-01)             │
│  Group         sel.rate  impact ratio (4/5ths = 0.80)                   │
│  Men           ████████ 0.42   1.00 (ref)                               │
│  Women         ███████▌ 0.39   0.93 ✓                                   │
│  Hispanic      ██████▌  0.34   0.81 ✓ │ 0.80 line ┄┄┄┄┄┄┄┄┄┄┄┄┄┄       │
│  Black         ██████   0.31   0.74 ⚠ below 4/5ths → review             │
│  Notice 10-biz-day ✓ all  ·  PII segmentation ✓  ·  [Export LL144 audit]│
└──────────────────────────────────────────────────────────────────────────┘
```

Notes:
- Purpose: Provide auditable proof that every decision was human-reviewable and fair — covering EU AI Act reasoning logs, time-travel replay, and Local Law 144 impact ratios.
- Key components/zones: (1) append-only reasoning-log viewer with model + timestamp per step (Wei Chen's 0.63 borderline routed to human, not auto-rejected); (2) LangGraph checkpoint scrubber with play/step controls showing state at each checkpoint; (3) bias-audit dashboard with selection rates, impact ratios, the 0.80 four-fifths line, candidate-notice status, and PII-segmentation indicator.
- Recruiter can DO: read the reasoning chain, scrub/step through checkpoints to replay state, inspect impact ratios, and export the LL144 audit (a flagged Black-group ratio of 0.74 triggers review).
- Real-time / SSE: `agent.thinking` / `task.completed` append new reasoning-log entries; `approval.required` surfaces here when a gate fires; impact ratios recompute as the cohort grows.
- Agents / back-end: Aegis (Opus, guardrail) owns COMP-EUAI-01/02, COMP-LL144-01/02 and COMP-PII-01; Lens (Sonnet) computes selection rates and impact ratios; the LangGraph checkpointer backs the replay scrubber; Echo executes the 10-business-day notice.

### 8.9 Analytics (Lens)

One-line purpose: Lens analytics dashboard — funnel conversion, time-to-stage, source/strategy effectiveness, recruiter hours saved, and a live what-if fairness simulator.

```
╔══════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                               🔔5   Riya R. ▾ ║
╠══════════════════════════════════════════════════════════════════════════╣
║ Pipeline │ Intake │ Approvals │ Sourcing │ [Analytics] │ Compliance      ║
╚══════════════════════════════════════════════════════════════════════════╝
┌── Lens · Req: [ REQ-2026-0431 Sr Backend Eng L5 ▾ ]  threshold 0.72 ─────┐
│ Funnel (Sourced → Offer)              │ Time-to-stage (median days)      │
│  Sourced   ██████████████████ 64      │ screen     ███░ 2.1d            │
│  Screened  ███████████░░░░░░░ 41      │ assessment █████░ 3.4d          │
│  Assessed  ███████░░░░░░░░░░░ 23      │ hir.mgr    ██████░ 4.0d         │
│  Onsite    ████░░░░░░░░░░░░░░ 11      │ values     ████░ 2.6d           │
│  Offer     ██░░░░░░░░░░░░░░░░  4       │ offer      ███░ 1.9d            │
├───────────────────────────────────────┼──────────────────────────────────┤
│ Source effectiveness (→Atlas A/B)     │ Recruiter hours saved            │
│  referral       ████████ 2 hires      │  this req  ████████████ 38h     │
│  internal_db    █████ 1 hire          │  Echo auto-replies  21h         │
│  linkedin       ██ 0 (in loop)        │  Sync scheduling    11h         │
│  job_board      █ 0                   │  Sift bulk screen    6h         │
├───────────────────────────────────────┴──────────────────────────────────┤
│ What-if fairness simulator   threshold ◀───────●──▶ 0.72                 │
│  at 0.72: pass 23 · impact ratio Black 0.74 ⚠   │ drag to 0.68:          │
│  → projected impact ratio 0.83 ✓ , +6 candidates advance  [Apply]        │
└──────────────────────────────────────────────────────────────────────────┘
```

Notes:
- Purpose: Show hiring-funnel health and fairness for a selected req, and let Riya model how a threshold change moves both yield and the impact ratio before committing.
- Key components/zones: req selector; Sourced→Offer funnel with counts (64→4); time-to-stage bars; source/strategy effectiveness feeding Atlas A/B routing (referral 2 hires leads); recruiter-hours-saved breakdown by agent; what-if fairness slider that live-recomputes impact ratio.
- Recruiter can DO: switch reqs, read the funnel/time/source charts, drag the threshold slider to preview fairness + yield effects, and [Apply] a new threshold (which routes through an approval gate).
- Real-time / SSE: `stage.changed` and `task.completed` increment funnel counts and time-to-stage; the simulator recomputes impact ratio on every slider drag; hours-saved ticks up as Echo/Sync/Sift complete tasks.
- Agents / back-end: Lens (Sonnet) computes all metrics and the fairness simulation; Atlas (Opus) consumes source effectiveness for A/B sourcing routing; Aegis validates that any applied threshold change preserves COMP-LL144-01 fairness and logs it.

### 8.10 Command Bar / Copilot & Notifications

One-line purpose: The Cmd-K natural-language copilot routed through Atlas, plus the live notifications stack of real A2A push events and approval pings.

```
╔══════════════════════════════════════════════════════════════════════════╗
║ TalentMesh   [ ⌕ search… ]                               🔔5   Riya R. ▾ ║
╠══════════════════════════════════════════════════════════════════════════╣
║ [Pipeline] │ Intake │ Approvals │ Sourcing │ Analytics │ Compliance      ║
╚══════════════════════════════════════════════════════════════════════════╝
        ┌── ⌘K Copilot · routed through Atlas (Opus) ──────────────────┐
        │ ❯ show me everyone stuck >5 days in screening                │
        │ ───────────────────────────────────────────────────────────│
        │ ◆ 2 candidates in screening > 5 days:                       │
        │   ⚠ Wei Chen   CAND-90120 · REQ-0455 · 0.63 borderline · 9d │
        │   ⚠ Marcus Webb CAND-90034 · REQ-0431 · 0.58 below · 10d    │
        │   [ Open Approvals ]  [ Message via Echo ]                   │
        │ Try: "advance Amara to offer" · "draft JD for REQ-0478"     │
        │      "compare strategy A vs B for REQ-0490"                  │
        └─────────────────────────────────────────────────────────────┘
                                                  ┌── 🔔 Notifications ──┐
                                                  │ ⧗ Sync (Calendar A2A)│
                                                  │  Amara Diallo onsite │
                                                  │  loop booked ✓  2m   │
                                                  │ ✓ Verify (BgCheck    │
                                                  │  A2A) Helena Brandt  │
                                                  │  bg-check complete 5m│
                                                  │ ▣ Bridge (IT A2A)    │
                                                  │  laptop ordered for  │
                                                  │  new hire  12m       │
                                                  │ ! Approval required: │
                                                  │  advance Daniel Osei │
                                                  │  → offer   [Review]  │
                                                  └──────────────────────┘
```

Notes:
- Purpose: Give Riya one keystroke (Cmd-K) to query the whole system in natural language and a passive stream of agent/A2A events so nothing falls through the cracks.
- Key components/zones: command-bar overlay with the NL query ("everyone stuck >5 days in screening") returning Wei Chen (9d) and Marcus Webb (10d) plus example commands; notifications/toast stack of real A2A push events (Sync onsite booked, Verify bg complete, Bridge laptop ordered) and an approval ping for Daniel Osei.
- Recruiter can DO: type any NL command, jump to results (Open Approvals, Message via Echo), run suggested commands, and act on toasts ([Review] an approval, dismiss).
- Real-time / SSE: `task.completed` drives the A2A toasts (CalendarCoordinator, BackgroundCheckProvider, ITProvisioning push notifications), `approval.required` raises the Daniel Osei ping, `agent.token` streams the copilot's answer.
- Agents / back-end: Atlas (Opus) parses and routes every command to the right sub-agent and aggregates results; Sync/Verify/Bridge (Haiku) emit the A2A push events via signed external agent cards; Aegis injects approval.required gates so no advancement is autonomous.

## 9. Capability catalog — everything the recruiter can do

This catalogs every action Riya (Technical Recruiter) and the secondary personas can take in the TalentMesh front end, grouped by area. Each capability names the screen it lives on (design §12) and the agent or system behind it. All concrete values are grounded in the synthetic data set.

### 9.1 Requisition & intake

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Start a req in plain language | Describe a role conversationally ("Senior Backend Engineer, payments, Austin hybrid, 5+ yrs"); no form-filling | Intake & JD Builder | Scribe (Sonnet 4.6) drafts intake |
| Review/edit AI-drafted JD | Side-by-side editor; inline edits to the JD-ENG-SENIOR template structure (one_line_mission → eeo_statement) | Intake & JD Builder | Scribe; LangGraph `interrupt()` |
| See inclusive-language / Aegis check | Inline banner flags `rockstar/ninja`, `aggressive`, `dominant`; confirms must-haves split from nice-to-haves to widen the funnel | Intake & JD Builder | Aegis (Opus 4.8, guardrail) per COMP-BIAS-01 |
| Approve & open req | One click resolves the LangGraph human-in-the-loop interrupt and opens the req (status `open`) | Intake & JD Builder | LangGraph durable thread |
| View comp band | See base/equity/bonus for the band, e.g. BAND-ENG-L5: $175k–$215k base, $90k equity, 15% bonus target | Intake & JD Builder / Candidate 360 | leveling_comp store |
| Set / adjust screening threshold | Set the auto-advance cutoff per req — 0.72 (REQ-0431), 0.70 (REQ-0455), 0.65 (REQ-0478), 0.78 (REQ-0490) | Intake & JD Builder | LangGraph `route_after_screen` |
| Set clearance & check requirements | Toggle `requires_clearance` (Secret for REQ-0490) and `requires_background_check` | Intake & JD Builder | Aegis / Verify gating |
| Clone a req | Duplicate an existing req (e.g. clone REQ-0478 for the 8-headcount high-volume hire) and adjust deltas | Pipeline / Mission Control | Scribe seeds from prior JD |

### 9.2 Pipeline management

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| View live kanban | Columns Sourced → Screened → Assessment → Interview → Checks → Offer; cards move without refresh | Pipeline / Mission Control | `stage.changed` SSE from LangGraph |
| Drag a card between stages | Manually nudge a candidate (e.g. move Daniel Osei from `hiring_manager` to next stage) | Pipeline / Mission Control | Atlas (Opus 4.8) re-routes; logged |
| Filter / sort | Filter by req, stage, status (`active`, `borderline`, `below_threshold`), source, location, clearance | Pipeline / Mission Control | Query layer |
| Bulk actions | Select many cards (high-volume REQ-0478, 200+/wk) and bulk-message or bulk-advance for approval | Pipeline / Mission Control | Echo / Atlas batch |
| Open Candidate 360 | Click a card to drill into one candidate's full record | Pipeline / Mission Control → Candidate 360 | — |
| See per-stage SLAs / aging | Card age vs the 3-business-day-response target (KB-FAQ-002); 2–3 week loop benchmark | Pipeline / Mission Control | Lens (Sonnet 4.6) |
| "Stuck >5 days" view | Saved view surfacing candidates idle past SLA for triage | Pipeline / Mission Control | Lens aging report |
| Agent Activity Feed | Plain-language stream ("Sift scored 38 resumes for REQ-0431", "Sync booked Amara's onsite") | Pipeline / Mission Control (right rail) | LangGraph streaming → `agent.thinking` SSE |

### 9.3 Sourcing

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Trigger Scout | Kick off multi-channel sourcing for a req | Pipeline / Mission Control | Scout (Sonnet 4.6, ParallelAgent) |
| Run semantic search w/ metadata filters | RAG search over candidate corpus with filters, e.g. `clearance=Secret` for REQ-0490 (surfaces Helena Brandt) | Pipeline / Mission Control | Chroma vector search, metadata-filtered |
| Compare A/B sourcing strategies | Compare internal_database vs linkedin vs referral / job_board channels | Analytics / Sourcing panel | Atlas A/B routing |
| View source effectiveness | Which channel yields hires (referral fast-tracked Amara 0.91 & Sofia 0.84; linkedin produced borderline Wei 0.63) | Analytics | Lens source-effectiveness artifact |
| Add to pipeline | Promote a sourced profile into a req's pipeline | Pipeline / Mission Control | Scout → Atlas |
| Dedupe | Detect the same person across channels before adding | Pipeline / Mission Control | Scout parallel merge |

### 9.4 Screening & ranking

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| View ranked shortlist | Candidates ranked by `screen_score` per req with sub-scores + plain-language explanations | Candidate 360 / Approvals Inbox | Sift (Haiku 4.5 → Sonnet 4.6, Sequential) |
| See sub-scores & explanations | Per-dimension breakdown tied to must-haves (Python, distributed systems, PostgreSQL, Kafka, AWS) | Candidate 360 | Sift |
| See threshold branch | Visual of advance / human_review / reject branch vs threshold (Wei 0.63 vs 0.70 → human_review; threshold−0.10 band) | Approvals Inbox | LangGraph `route_after_screen` |
| Approve / override advance | Approve a recommended advancement; AI never advances autonomously | Approvals Inbox | Aegis (COMP-EUAI-02) |
| Approve / override reject | No agent auto-rejects; human confirms (Marcus Webb 0.58, Jordan Lee 0.49 below threshold) | Approvals Inbox | Aegis guardrail |
| Request a borderline human review | Pull a borderline candidate (Wei Chen, `status: borderline`) into manual judgment instead of auto-decision | Approvals Inbox | LangGraph interrupt |
| See reasoning-log excerpt | The append-only rationale snippet attached to each approval item | Approvals Inbox | `reasoning_log` (EU AI Act COMP-EUAI-01) |

### 9.5 Engagement / communication

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Message a candidate via Echo | Compose / hand a message to the candidate, RAG-grounded in company_knowledge | Candidate 360 | Echo (Sonnet 4.6 + Haiku 4.5) |
| See full interaction history | Long-term memory timeline (e.g. Priya: outreach → benefits chat → A2A scheduling) | Candidate 360 | Echo memory store |
| Send / approve outreach | Approve personalized outreach (Amara's PCI-tokenization warm intro; Helena's FedRAMP reference) | Approvals Inbox / Candidate 360 | Echo; Aegis pre-send bias scan (COMP-BIAS-01) |
| Trigger / inspect nurture loop | Re-engage rejected candidates (Jordan Lee flagged to re-engage in 90 days, CRM micro-course) | Candidate 360 | Echo LoopAgent nurture |
| Switch channel (web/SMS/email) | Respect candidate channel preference (Daniel Osei → SMS) | Candidate 360 / candidate chat | Echo channel-aware |
| Escalate chat to video | Promote a chat thread to a 15-min video call (Sofia's UX negotiation) | Candidate 360 | Echo → Sync; IL VIA consent (COMP-ILVIA-01) |
| See anti-ghosting flags | Surfaces candidates owed a status update vs the "always respond, even if no" promise (KB-FAQ-002) | Candidate 360 / Pipeline | Echo + Lens |

### 9.6 Assessment

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Issue an assessment | Send the role-specific assessment (system design / situational / values) | Candidate 360 | Gauge (Opus 4.8, LoopAgent) |
| View Gauge scoring vs rubric | Weighted scoring against the rubric, e.g. RUB-ENG-SYSDESIGN (scalability 0.30, decomposition 0.25, tradeoff 0.25, comms 0.20), pass 0.70 | Candidate 360 | Gauge |
| See iterate-until-pass loop | The `max_iterations=3` loop refining the assessment until the rubric is satisfied | Candidate 360 | Gauge LoopAgent |
| Review live coding / voice session | Watch a live session stream (tokens render live) | Candidate 360 | Gauge `astream()`; `agent.token` SSE |
| Mark pass/fail recommendation | Record a pass/fail recommendation, e.g. Amara `assessment_score` 0.88, Sofia 0.79, Daniel 0.76 (CX pass 0.65) | Candidate 360 / Approvals Inbox | Gauge → human approval |

### 9.7 Scheduling

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Request mutual slots across N interviewers | Find slots across the full loop (Amara's onsite = 5 interviewers, multi-party) | Candidate 360 | Sync (Haiku 4.5) → CalendarCoordinator `find_mutual_slots` |
| View proposed slots | See returned candidate options | Candidate 360 | CalendarCoordinator |
| Confirm booking | Book the event and send invites | Candidate 360 | `book_interview` (A2A) |
| Reschedule | Move a booking and notify all parties | Candidate 360 | `reschedule` (A2A) |
| See A2A task status + push | Live task state with push on completion (supports_push_notifications: true) | Candidate 360 | CalendarCoordinator A2A push → `task.completed` SSE |

### 9.8 Checks

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Trigger bg + reference in parallel (with consent) | Launch background + reference checks together on a consent token | Candidate 360 | Verify (Haiku 4.5 + Sonnet 4.6, Parallel) |
| View A2A task status | Track BackgroundCheckProvider (`long_running: true`) and ReferenceCheckAgent state | Candidate 360 | A2A `get_status` |
| Read summarized report | Read the synthesized check summary | Candidate 360 | Verify (Sonnet) → `fetch_report` |
| Gate offer on result | Block offer until checks clear (Amara `background_check_status: passed`; Helena `in_progress`; mandatory & gated for REQ-0490) | Candidate 360 / Approvals Inbox | Aegis offer gate |

### 9.9 Offer & decision

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Assemble decision packet | One view: screen score + assessment + checks + interview rubric scores | Approvals Inbox / Candidate 360 | Atlas aggregates |
| Approve / override offer | Human-approved offer; AI cannot auto-hire | Approvals Inbox | Aegis (COMP-EUAI-02) |
| See comp recommendation within band | Recommended base/equity inside the band (e.g. L5 eng within $175k–$215k; L6 within BAND-ENG-L6 $215k–$265k) | Approvals Inbox | leveling_comp; benchmarked to 75th pct (KB-BEN-002) |

### 9.10 Onboarding handoff

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Trigger HRIS record on accept | On offer accept, create the employee record | Candidate 360 | Bridge (Haiku 4.5) → HRISOnboarding `create_employee` |
| Trigger IT provisioning | Order laptop/licenses/access by role profile | Candidate 360 | Bridge → ITProvisioning `provision_by_role` |
| Watch parallel push status | Watch both HRIS + IT pushes complete in parallel | Candidate 360 | A2A push → `task.completed` SSE |

### 9.11 Governance & audit

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Open reasoning log | Append-only chain of why each decision was made, per candidate | Compliance & Audit | `reasoning_log` (COMP-EUAI-01) |
| Replay / time-travel a decision | Re-run the graph to any prior checkpoint to show exact state behind a decision | Compliance & Audit | LangGraph time travel |
| View impact-ratio dashboard | Selection rates and impact ratios across sex and race/ethnicity (and intersections) | Compliance & Audit | Lens (COMP-LL144-01) |
| Export LL144 audit | Export the bias-audit artifact for the annual independent AEDT audit | Compliance & Audit | Lens export |
| See candidate-notice + alternative-process status | Per-candidate 10-business-day notice status and any alternative-process request | Compliance & Audit / Candidate 360 | Aegis + Echo (COMP-LL144-02) |
| PII-segmentation indicator | Confirms demographic data never reached scoring/ranking agents | Compliance & Audit | Aegis (COMP-PII-01) |

### 9.12 Analytics

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Funnel conversion | Stage-to-stage conversion across the funnel | Analytics | Lens (Sonnet 4.6) |
| Time-to-stage | Aging per stage vs target_fill_date (e.g. REQ-0431 → 2026-07-15) | Analytics | Lens |
| Source effectiveness | Which channel/strategy yields hires; feeds Atlas A/B routing | Analytics | Lens |
| Hours saved | Recruiter-hours-saved estimate | Analytics | Lens |
| What-if fairness simulator | Simulate threshold/criteria changes against impact ratios before applying | Analytics / Compliance & Audit | Lens + Aegis |

### 9.13 Productivity

| Capability | Description | Where in the UI | Agent / system behind it |
|---|---|---|---|
| Command bar / copilot | Natural-language queries ("show stuck REQ-0478 candidates", "who's blocked on checks") | Global command bar | Atlas (Opus 4.8) |
| Notifications / toasts | Real-time toasts on `approval.required` and `task.completed` (e.g. bg-check done) | Global | SSE → toast |
| Saved views | Persist filtered board views (e.g. "stuck >5 days", "borderline") | Pipeline / Mission Control | Query layer |
| Keyboard shortcuts | Fast approve/override, navigate cards, open Candidate 360 | Global | Front end |

## 10. Roles & permissions

Access is role-based. The four roles map to the personas: **Recruiter** (Riya), **Hiring Manager** (Marcus), **Compliance** (Dana), and **Admin** (platform owner; Theo configures it). Legend: **Full** = view + act/edit; **Approve-only** = can approve/override but not edit underlying config; **View** = read-only; **None** = not accessible.

| Capability area | Recruiter (Riya) | Hiring Manager (Marcus) | Compliance (Dana) | Admin |
|---|---|---|---|---|
| Requisition & intake (start/edit JD, open req) | Full | Full (initiates intake, co-edits JD) | View | Full |
| Set/adjust screening threshold | Approve-only (proposes) | View | View | Full (governs thresholds) |
| Comp band view/edit | View | View | None | Full (manages bands) |
| Pipeline management (kanban, drag, bulk) | Full | View + nudge own reqs | View (read-only) | Full |
| Sourcing (Scout, semantic search, A/B) | Full | View shortlist | View | Full |
| Screening & ranking (shortlist, sub-scores) | Full | View shortlist (own reqs) | View | Full |
| Approve/override advance | Approve-only | Approve-only (own reqs) | None | Approve-only (break-glass) |
| Approve/override reject | Approve-only | Approve-only (own reqs) | None | Approve-only (break-glass) |
| Borderline human review | Full | Approve-only | View | View |
| Engagement / candidate messaging (Echo) | Full | View history | View | Full |
| Nurture loop | Full | None | View | Full |
| Assessment (issue, Gauge vs rubric) | Full | View results + score | View | Full (rubric config) |
| Scheduling (Sync / A2A) | Full | Confirm own slots | None | Full (vendor config) |
| Checks (Verify bg + reference) | Full (with consent) | View result only | View (full reports) | Full (vendor config) |
| Offer & decision packet | Full (assembles) | Approve-only (offer approval) | View | Approve-only |
| Onboarding handoff (Bridge HRIS/IT) | Full | View status | None | Full (integration config) |
| Reasoning-log viewer | View | View (own reqs) | Full | Full |
| Replay / time-travel | View | None | Full | Full |
| Impact-ratio / bias dashboard | View | None | **Full** | View |
| LL144 audit export | None | None | **Full** | Full |
| What-if fairness simulator | View | None | **Full** | Full |
| Candidate-notice / alternative-process status | View | None | Full (owns) | Full |
| PII-segmentation indicator | View | None | Full | Full |
| Analytics (funnel, source, hours saved) | Full | View (own reqs) | View | Full |
| User / role management | None | None | None | **Full** |
| Integration / A2A vendor management | None | None | None | **Full** |

Key boundaries, deliberately realistic:
- **Hiring Manager** participates where the role demands it — intake, shortlist review, and **offer approval** for their own reqs — but has **no access** to the bias dashboard, LL144 export, time-travel, or any admin config.
- **Compliance (Dana)** has **full** audit, bias dashboards, and what-if simulation, but **cannot edit the pipeline**, advance/reject candidates, or message candidates — oversight without operational control, preserving auditor independence.
- **Admin** manages users, integrations, and thresholds, but advance/reject/offer remain **approve-only break-glass** even for Admin, so no role can silently bypass the human-decision boundary.

**Delegation of human-decision-boundary approvals.** The EU AI Act human-oversight requirement (COMP-EUAI-02) is satisfied by an Approvals Inbox item that requires exactly one accountable human per advancement, rejection, and offer. Approval authority can be **delegated** between Recruiter and Hiring Manager per req (e.g. Riya delegates offer approval on REQ-0431 to Marcus, or covers for a Hiring Manager on PTO). Delegation is itself a logged event: the reasoning_log records the **delegator, delegate, scope, and time window**, and the resulting approval is attributed to the human who actually clicked — never to the agent and never to the absent delegator. Compliance can therefore always answer "which trained person reviewed and approved this decision?" for any candidate, which is exactly the reviewability the EU AI Act demands. Delegation never lowers the bar to fewer than one human approver.

## 11. States, edge cases & empty/error states

Non-happy-path UI states, what Riya (or Dana) sees, and the recovery action. Grounded in the live data set.

| State / edge case | What the recruiter sees | Recovery action |
|---|---|---|
| **Empty pipeline (new req, no candidates)** | Freshly opened req (e.g. REQ-0490 opened 2026-06-12) shows empty kanban columns with a "No candidates yet" placeholder and a prominent **Trigger Scout** call-to-action; Agent Activity Feed idle | One click runs Scout (ParallelAgent) across channels; empty state replaced by streaming "sourcing…" rows |
| **Scout returns too few candidates** | Banner: "Only N profiles met must-haves" — e.g. the clearance-gated REQ-0490 corpus is thin | Atlas **fallback strategy**: relax nice-to-haves, widen location/seniority, or add a channel, then re-run Scout; recruiter approves the broadened search before it fires |
| **Borderline score routed to human** | Wei Chen surfaces in Approvals Inbox: `0.63 vs 0.70 threshold`, `status: borderline`, with sub-scores + reasoning-log excerpt and **Approve / Override / Request changes** (no auto-reject) | Recruiter makes the call; decision appended to reasoning_log; card advances or moves to nurture |
| **Candidate withdraws / goes silent** | Card flagged with a **ghost-risk** indicator when past the 3-business-day response promise (KB-FAQ-002); interaction history shows last contact | Echo fires the **nurture loop** (as for Jordan Lee: re-engage in 90 days + CRM micro-course); recruiter can escalate channel (e.g. email → SMS) |
| **Candidate requests alternative process (LL144)** | Candidate-360 + Compliance banner: "Alternative selection process requested" with the AEDT-notice status (COMP-LL144-02, KB-FAQ-004) | Route to a human-only review path; AI screening paused for that candidate; Aegis logs the request and the alternative provided |
| **A2A vendor task fails / times out (calendar / bg-check)** | Candidate-360 task chip flips to **Failed / Timed out** instead of `task.completed`; toast surfaces the error | Retry the A2A `send_task`; on repeated failure, fall back to manual scheduling or an alternate vendor; durable thread keeps the rest of the loop intact |
| **Background check returns a flag** | Verify report summary shows a **flag** rather than clear (contrast Amara `passed`); offer gate stays closed | Recruiter/Compliance reviews the summarized report; offer remains **gated** (mandatory for REQ-0490) until human adjudicates; decision logged |
| **Version-negotiation fallback (reference vendor v0.3)** | ReferenceCheckAgent card shows "negotiated 1.0 → **0.3** (compatibility shim)"; reference task runs with reduced features, **no push** (supports_push_notifications: false) | System **polls** for status instead of awaiting push; recruiter sees a "polling" indicator; if shim fails, fall back to manual reference collection |
| **Clearance filter excludes a candidate** | On REQ-0490, a strong profile lacking `clearance=Secret` is filtered out of the candidate_corpus retrieval with an explainer ("excluded: clearance metadata"); Helena Brandt (Secret) remains | Recruiter can route the excluded person to a **non-clearance req** instead of overriding the legal requirement; exclusion logged |
| **Agent run errors / graph paused / resume-after-crash** | Banner: "Run paused — will resume"; card freezes at its last checkpoint, no data loss | **Durable execution** (PostgresSaver checkpointer) resumes the LangGraph thread from the last checkpoint after recovery; a req can pause for weeks and resume cleanly |
| **Approval left pending too long (SLA breach)** | Approvals Inbox item turns red past SLA; "stuck >5 days" view and aging badge highlight it | Reassign or **delegate** the approval (Recruiter ↔ Hiring Manager); reminder push sent; SLA breach recorded for analytics |
| **Offline / connection-lost during live stream** | Live coding/voice session or token stream shows "Reconnecting…"; kanban stops receiving `stage.changed` | On reconnect, SSE/WebSocket resubscribes and the durable state **re-syncs** the board to current truth; the underlying graph kept running server-side, so no progress is lost |

## 12. Non-functional requirements

The recruiter app must meet the following non-functional requirements, grouped by concern. Targets are set against the demo data scale (4 open reqs, the 200+/week high-volume role REQ-2026-0478) and the mission-control interaction model.

### 12.1 Real-time

| Requirement | Detail | Target |
|---|---|---|
| Mission-control activity feed | Stream `agent.thinking`, `agent.token`, `stage.changed`, `task.completed` over **SSE** (native `EventSource`) into the Pipeline / Mission Control feed | First token < 1s; subsequent tokens streamed incrementally |
| Approval signaling | `approval.required` events surface a new item in the Approvals Inbox without a page refresh | Inbox badge updates < 2s after event emission |
| Kanban live updates | Stage changes (e.g., Amara Diallo moving to `values_interview`) move cards on the board reactively via `stage.changed` | No manual refresh; optimistic UI reconcile via TanStack Query |
| Candidate chat | Bidirectional candidate Q&A (Echo) over a **WebSocket** channel, token-streamed | p95 round-trip for a streamed reply start < 1.5s |

### 12.2 Performance & scale

| Requirement | Detail | Target |
|---|---|---|
| High-volume intake | REQ-2026-0478 (Customer Support Specialist, headcount 8) expects **200+ applications/week**; Sift must parse/extract on Haiku 4.5 at volume | Screen thousands of resumes without backlog; per-resume parse latency low enough to clear a weekly batch within hours |
| Concurrent requisitions | Support **12–18 active reqs per recruiter** concurrently (REC-08 already carries REQ-2026-0431 and REQ-2026-0455) | No degradation of feed/board responsiveness at the top of that range |
| Page load | Pipeline, Candidate 360, Approvals, Compliance, Analytics screens | p95 page load < 2.5s |
| Parallel sourcing | Scout (ParallelAgent) fans out to internal corpus, LinkedIn (MCP), and referrals simultaneously, then de-duplicates | Channels run concurrently, not serially |

### 12.3 Reliability & durability

- **Durable LangGraph execution.** A requisition stays open for weeks (REQ-2026-0490 target fill 2026-08-30); the graph must persist state across that lifespan.
- **Resume-after-crash.** A **Postgres checkpointer** holds per-req/per-candidate thread state; a paused or crashed run resumes exactly where it left off, with no lost reasoning-log entries.
- **A2A long-running tasks with push.** Background checks are long-running (BackgroundCheckProvider `long_running: true`, supports push); the system must tolerate hours-to-days task durations and resume on push completion rather than blocking (Helena Brandt's `background_check_status: in_progress` must survive restarts).

### 12.4 Security & privacy

- **AuthN / AuthZ.** OAuth2/OIDC sign-in (Auth.js); **RBAC** with distinct roles — Recruiter (Riya), Hiring Manager (Marcus), Compliance/DEI (Dana), Admin/Platform (Theo).
- **PII segmentation gate.** Enforce COMP-PII-01: demographic data used for bias auditing is segmented at the architecture level and **must never reach scoring or ranking agents**; scoring agents see only job-relevant attributes.
- **Signed A2A agent cards.** All five vendor cards are `signed: true`; Sync, Verify, and Bridge must verify the card signature against its `signature_domain` before trusting a vendor.
- **OAuth2 scopes per vendor.** Enforce least-privilege per card — e.g., `calendar.read`/`calendar.write` (CalendarCoordinator), `bgcheck.initiate`/`bgcheck.status` (BackgroundCheckProvider), `employee.create`/`onboarding.trigger` (HRISOnboarding), `provisioning.create` (ITProvisioning); ReferenceCheckAgent uses an `api_key` scheme.
- **Multi-tenancy isolation.** TalentMesh is SaaS — one deployment serves many client companies; each tenant's data, agents, and threads must be isolated.

### 12.5 Accessibility

- Conform to **WCAG 2.1 AA**.
- Full **keyboard navigation** across the kanban board, Approvals Inbox, and Candidate 360.
- **Screen-reader labels** on the live agent activity feed (announce `agent.thinking` / `stage.changed` / `approval.required` events as accessible status updates, not silent DOM churn).

### 12.6 Auditability

- **Append-only reasoning log** for every ranking/advancement/rejection decision (COMP-EUAI-01).
- **Time-travel / replay** of any candidate's full decision path from the Compliance & Audit screen.
- **Exportable** audit records (reasoning log + replay trace) for the LL144 annual independent bias audit and regulator review.

### 12.7 Observability

- **LangSmith tracing** of agent trajectories across the full funnel, plus evaluation of decision quality and production observability for Theo.

### 12.8 Internationalization

- v1.0 ships **US-English only**. Multi-language candidate surfaces and localization are noted as a **future extension** (see Section 15) — design copy and data layers so locale can be added without schema changes.

## 13. Data model & grounding

This is the "what the demo runs on" reference. The data files ship a **full synthetic library** (the at-a-glance table below); the per-entity tables that follow it list the **canonical records** the §8 wireframes are built against — a stable subset of the larger library so screen examples stay consistent and reproducible.

### 13.0 Data library at a glance (full file contents)

| File | Full library | Canonical subset (used in §8 wireframes) |
|---|---|---|
| `jobs.json` | **100** requisitions across 14 role families, IC L1–L7 / mgmt M3–M7, US + international, realistic thresholds/headcount/clearance | the 4 reqs in §13.1 |
| `candidates.json` | **1,000** candidates with realistic funnel distribution (~36% below-threshold, ~12% borderline, ~51% active, hired); resume text, sub-scores, stages, interaction history | the 8 in §13.2 |
| `leveling_comp.json` | **508** comp bands (family × level × tier), 12 levels, 14 JD templates, role families, location tiers, leveling competencies, comp philosophy | the 4 bands in §13.6 |
| `company_knowledge.json` | **316** KB docs across 17 categories (benefits, culture, FAQ, policy, perks, offices, interview prep, immigration, onboarding, compensation, security/IT, DEI, L&D, wiki/glossary, process, tools, role guides) | Echo's `company_knowledge` RAG corpus |
| `compliance_rules.json` | **33** rules across 13 jurisdictions (EU AI Act, GDPR, NYC LL144, CO/IL/CA/TX/UT/MD/NJ, UK, Canada) + **4** frameworks (NIST AI RMF, ISO 42001/23894, SOC2) + **7** data-subject rights | the 8 in §13.4 |
| `interview_rubrics.json` | **17** weighted rubrics across role families + cross-cutting stages, anchored 1–5 scales, bias-mitigation notes | the 3 in §13.5 |
| `external_agent_cards.json` | **17** A2A vendor cards, all **locally runnable** (`local_url`/`local_port`/`mock_command`/`health_check` + a `local_dev` registry) | the 5 in §13.3 |

### 13.1 Requisitions — canonical demo reqs (4 of 100 in `jobs.json`)

| Req ID | Title | Level | Location | Threshold | Headcount | Clearance | BG check |
|---|---|---|---|---|---|---|---|
| REQ-2026-0431 | Senior Backend Engineer | L5 | Austin, TX (Hybrid) | 0.72 | 2 | — | Required |
| REQ-2026-0455 | Product Designer (Senior) | L5 | Remote (US) | 0.70 | 1 | — | Required |
| REQ-2026-0478 | Customer Support Specialist | L2 | Phoenix, AZ (On-site) | 0.65 | 8 | — | Required |
| REQ-2026-0490 | Staff Security Engineer | L6 | Washington, DC (Hybrid) | 0.78 | 1 | Secret | Required (gated before offer) |

> REQ-2026-0478 is the high-volume role (200+ apps/week → Parallel sourcing + Sequential screening). REQ-2026-0490 requires `clearance = Secret` metadata filtering on candidate retrieval.

### 13.2 Candidates — canonical demo set (8 of 1,000 in `candidates.json`)

| Candidate ID | Name | Req | Stage | Screen score | Status | Source | Notable flag |
|---|---|---|---|---|---|---|---|
| CAND-90105 | Amara Diallo | REQ-2026-0431 | values_interview | 0.91 | active | referral (EMP-204) | Top score; bg check **passed** |
| CAND-90012 | Priya Nair | REQ-2026-0431 | system_design | 0.86 | active | internal_database | Above threshold |
| CAND-90034 | Marcus Webb | REQ-2026-0431 | screening | 0.58 | below_threshold | linkedin | Below 0.72; talent-pool for L4 |
| CAND-90051 | Sofia Alvarez | REQ-2026-0455 | design_exercise | 0.84 | active | referral (EMP-771) | UX-negotiated video call (A2A) |
| CAND-90120 | Wei Chen | REQ-2026-0455 | screening | 0.63 | **borderline** | linkedin | 0.63 vs 0.70 → human review |
| CAND-90067 | Daniel Osei | REQ-2026-0478 | hiring_manager | 0.82 | active | job_board | Bilingual; SMS-preferred channel |
| CAND-90099 | Jordan Lee | REQ-2026-0478 | screening | 0.49 | below_threshold | job_board | Below 0.65; 90-day nurture loop |
| CAND-90081 | Helena Brandt | REQ-2026-0490 | leadership_panel | 0.88 | active | internal_database | **Secret** clearance; bg **in_progress** |

### 13.3 A2A vendor agents — canonical 5 of 17 (`external_agent_cards.json`, locally runnable)

| Vendor agent | Internal agent | Key skill(s) | Version | Signed | Push |
|---|---|---|---|---|---|
| CalendarCoordinator | Sync | `find_mutual_slots`, `book_interview`, `reschedule` | 1.0 | Yes | Yes |
| BackgroundCheckProvider | Verify | `initiate_check`, `get_status`, `fetch_report` (long-running) | 1.0 | Yes | Yes |
| ReferenceCheckAgent | Verify | `request_references` | 0.3 (negotiated down from 1.0) | Yes | No |
| HRISOnboarding | Bridge | `create_employee`, `trigger_onboarding` | 1.0 | Yes | Yes |
| ITProvisioning | Bridge | `provision_by_role` | 1.0 | Yes | Yes |

> The full file expands these to **17 cards** — adding SourcingMarketplace (Scout), VideoInterviewProctor (Gauge, IL-VIA consent), CodingAssessmentSandbox (Gauge), SkillsTestProvider (Sift/Gauge), IdentityVerification & Right-to-Work/I-9 (Verify), PayrollProvider (Bridge), EsignatureService (offers), SmsEmailComms (Echo), AssessmentBiasAuditor (Aegis/Lens), JobBoardDistributor (Scribe/Scout), and GlobalScreening (Verify). Every card is runnable locally via its `local_url` + `mock_command`, discovered through a `local_dev` registry (`python -m talentmesh.mock_a2a --all`) — so the whole A2A ecosystem can be exercised offline (see `agent.md` Phase 7).

### 13.4 Compliance rules — canonical 8 of 33 (`compliance_rules.json`)

| Rule ID | Regulation | Requirement (one-liner) | Enforced by | Blocking? |
|---|---|---|---|---|
| COMP-EUAI-01 | EU AI Act 2024/1689 | Keep a reviewable, overridable reasoning log for every ranking/advancement decision | Aegis | Yes |
| COMP-EUAI-02 | EU AI Act | A human must approve every advancement and every rejection; AI never decides autonomously | Aegis | Yes |
| COMP-LL144-01 | NYC Local Law 144 | Compute selection rates and impact ratios across sex/race for the annual bias audit | Aegis (+ Lens) | No |
| COMP-LL144-02 | NYC Local Law 144 | Notify applicants ≥10 business days before AEDT use; offer an alternative process | Aegis (+ Echo) | Yes |
| COMP-ILVIA-01 | Illinois AI Video Interview Act | Obtain consent and explain the AI before any AI analysis of video | Aegis | Yes |
| COMP-EEOC-01 | EEOC / Title VII | Screening criteria must be job-related; flag disproportionate filtering of protected groups | Aegis | No |
| COMP-BIAS-01 | Internal fairness policy | Scan JDs and candidate messages for biased/exclusionary language before send/publish | Aegis (+ Scribe) | Yes |
| COMP-PII-01 | Internal data governance | Segment demographic data architecturally; it must never reach scoring/ranking agents | Aegis | Yes |

> The full file expands these to **33 rules** spanning 13 jurisdictions (adding EU AI Act FRIA/transparency, GDPR Art. 22 + DPIA, Colorado AI Act, Illinois HB 3773, California FEHA ADS, Texas TRAIGA, Utah, Maryland, New Jersey, UK, Canada, plus ADA/ADEA/GINA/OFCCP), **4** governance frameworks (NIST AI RMF, ISO/IEC 42001, ISO/IEC 23894, SOC2), and **7** candidate data-subject rights (access, deletion, explanation, opt-out, human review). Aegis enforces them; Lens/Echo/Scribe support specific rules.

### 13.5 Interview rubrics — canonical 3 of 17 (`interview_rubrics.json`)

| Rubric ID | Loop stage | Role family | Pass threshold |
|---|---|---|---|
| RUB-ENG-SYSDESIGN | system_design | engineering | 0.70 |
| RUB-CX-SITUATIONAL | situational_assessment | customer_experience | 0.65 |
| RUB-VALUES | values_interview | all | 0.72 |

> The full file expands these to **17 rubrics** — coding screen, system design, eng management, data/ML, product sense + execution, design portfolio + exercise, security threat-modeling, CX situational, sales, marketing, finance, people/recruiting, legal, recruiter screen, hiring-manager, and the values interview — each with weighted dimensions, anchored 1–5 behavioral scales, interviewer guidance, and bias-mitigation notes, following 2025–2026 structured-interview best practice ("culture add, not culture fit").

### 13.6 Comp bands — canonical 4 of 508 (`leveling_comp.json`)

| Band ID | Family | Level | Base min | Base max | Equity/yr | Bonus target |
|---|---|---|---|---|---|---|
| BAND-ENG-L5 | engineering | L5 | $175,000 | $215,000 | $90,000 | 15% |
| BAND-ENG-L6 | engineering | L6 | $215,000 | $265,000 | $150,000 | 20% |
| BAND-DESIGN-L5 | design | L5 | $160,000 | $195,000 | $75,000 | 12% |
| BAND-CX-L2 | customer_experience | L2 | $52,000 | $68,000 | $8,000 | 8% |

## 14. Acceptance criteria

Testable criteria for v1.0 of the recruiter app, organized by screen/flow. Each is verifiable against the seeded demo data.

**Intake & JD Builder**
- [ ] Given Marcus requests "two senior backend engineers for payments," when Scribe drafts the JD, then it is leveled to L5, references BAND-ENG-L5, and lists must-haves separately from nice-to-haves.
- [ ] Given a JD draft contains "rockstar," "ninja," "aggressive," or "dominant," when Aegis scans it (COMP-BIAS-01), then publishing is **blocked** and the offending terms are flagged for the recruiter to fix.
- [ ] Given a JD passes the bias scan, when it is ready, then it is **not** published until Riya gives explicit human-in-the-loop approval.

**Pipeline / Mission Control**
- [ ] Given an agent emits a `stage.changed` event (e.g., Amara → values_interview), when it arrives over SSE, then the candidate's kanban card moves stages without a page refresh.
- [ ] Given an agent is working, when it streams `agent.thinking`/`agent.token`, then the mission-control feed renders the activity live and announces it to screen readers.
- [ ] Given REC-08 has both REQ-2026-0431 and REQ-2026-0455 open, when Riya loads the board, then both reqs render with their candidates and the page reaches p95 < 2.5s.

**Approvals Inbox**
- [ ] Given Wei Chen scores 0.63 against REQ-2026-0455's 0.70 threshold, when screening completes, then Wei appears in the Approvals Inbox as a **borderline human-review** item and is **never auto-rejected**.
- [ ] Given Marcus Webb scores 0.58 against REQ-2026-0431's 0.72 threshold, when screening completes, then no rejection is sent until a human approves it (COMP-EUAI-02).
- [ ] Given Amara Diallo scores 0.91, when she is recommended for advancement, then advancement still requires a human approval click and is not auto-executed.
- [ ] Given an `approval.required` event fires, when it arrives, then the Approvals Inbox badge updates within 2s without a refresh.
- [ ] Given any approval decision, when Riya approves or overrides, then the action and rationale are written to the append-only reasoning log.

**Sourcing**
- [ ] Given REQ-2026-0490 requires Secret clearance, when Scout sources candidates, then the candidate retrieval applies a `clearance = Secret` metadata filter and surfaces Helena Brandt (CAND-90081).
- [ ] Given REQ-2026-0478 receives 200+ applications/week, when Scout runs, then it fans out to internal corpus, LinkedIn, and referrals **in parallel** and de-duplicates the results.

**Candidate 360**
- [ ] Given Priya Nair's profile, when Riya opens Candidate 360, then it shows screen_score 0.86, assessment_score 0.81, current stage system_design, and the full `interaction_history` timeline (Echo + Sync touches).
- [ ] Given Helena Brandt's profile, when viewed, then her Secret clearance and `background_check_status: in_progress` are displayed prominently.
- [ ] Given Daniel Osei prefers SMS, when an update is sent, then the channel preference is respected and reflected in his interaction history.

**Engagement (Echo)**
- [ ] Given Jordan Lee scores 0.49 (below 0.65), when rejected, then an encouraging message with a CRM micro-course link is sent and the candidate is flagged for a 90-day nurture re-engagement loop.
- [ ] Given a candidate asks about benefits or visa in chat, when Echo replies, then the answer is grounded in `company_knowledge` RAG and streamed token-by-token over WebSocket.
- [ ] Given any candidate before an AEDT is used, when Echo communicates, then the required 10-business-day LL144 notice and the request-an-alternative option are surfaced (COMP-LL144-02).
- [ ] Given a text screening chat needs a richer conversation, when Echo escalates, then an A2A UX negotiation offers a video call.

**Scheduling (Sync)**
- [ ] Given Amara's 5-interviewer onsite loop, when Sync schedules, then it calls CalendarCoordinator over A2A to find mutual slots across all parties and books the events.
- [ ] Given a vendor agent card, when Sync connects, then it verifies the card's signature against its `signature_domain` before trusting it.

**Checks (Verify)**
- [ ] Given Helena's checks, when Verify runs, then it initiates BackgroundCheckProvider and ReferenceCheckAgent **concurrently** and tolerates long-running task durations via push notifications.
- [ ] Given ReferenceCheckAgent only speaks v0.3, when Verify connects requesting 1.0, then version negotiation succeeds down to 0.3 with a compatibility shim.

**Offer & onboarding (Bridge)**
- [ ] Given REQ-2026-0490 is clearance-gated, when an offer is prepared, then the offer is blocked until the background check passes (Helena's `in_progress` cannot proceed to offer).
- [ ] Given an offer is accepted, when Bridge runs, then it calls HRISOnboarding (`create_employee`) and ITProvisioning (`provision_by_role`) in parallel, with push notifications back to the recruiter.
- [ ] Given any offer, when prepared, then base/equity/bonus stay within the role's comp band (e.g., BAND-ENG-L6 base $215k–$265k for REQ-2026-0490).

**Compliance & Audit**
- [ ] Given any candidate, when Dana opens the audit view, then she can time-travel/replay the full decision path and export the reasoning log (COMP-EUAI-01).
- [ ] Given a demographic attribute exists for bias auditing, when scoring runs, then the PII-segmentation gate guarantees that attribute never reaches a scoring/ranking agent (COMP-PII-01).
- [ ] Given a must-have criterion could disparately filter a protected group, when Aegis reviews, then it is flagged for job-relatedness (COMP-EEOC-01).
- [ ] Given any AI-analyzed video interview, when it is initiated, then IL VIA consent is captured first (COMP-ILVIA-01).

**Analytics (Lens)**
- [ ] Given the current pipeline, when Lens runs the bias audit, then it computes selection rates and impact ratios across protected groups for the LL144 annual audit.
- [ ] Given Dana adjusts a screening threshold, when she runs the what-if, then the resulting impact-ratio change is shown (where the simulator is available) before any policy change.

## 15. Future extensions (post-v1.0)

Roadmap items beyond v1.0, each with its recruiter value:

- **Internal mobility / talent marketplace.** Reuse the candidate corpus over *employees* to match internal candidates to open reqs and build learning plans — lets Riya fill roles from existing staff before going external.
- **Voice screening agent.** A fully bidirectional-streaming voice interviewer (ADK live) for Tier-1 volume roles like REQ-2026-0478, with Illinois-VIA consent built in — automates first-round screening at 200+/week scale.
- **Offer-modeling agent.** Generates competitive offers within comp-band guardrails and models accept-probability — speeds offer construction and reduces declines.
- **Sentiment / candidate-experience (ghost-risk) monitor.** Watches engagement signals to flag likely-to-ghost candidates for proactive outreach — protects the recruiter's at-risk pipeline.
- **"Recruiter copilot" command bar.** Natural-language commands ("show me everyone stuck >5 days in screening") routed through Atlas — turns ad-hoc questions into instant pipeline actions.
- **Skills-gap & workforce-planning agent.** Aggregates pipeline plus headcount to forecast hiring needs — helps Riya and leadership plan future reqs proactively.
- **Inbound TalentMesh-as-an-A2A-server.** Expose TalentMesh agents to partners via the existing A2A server facade so external sourcing marketplaces can push candidates in — widens the top of the funnel automatically.
- **Multi-language + localization.** Localized candidate surfaces for global pools — lets Riya recruit beyond US-English candidates.
- **Fairness "what-if" simulator.** Lets Dana adjust a threshold and instantly see the impact-ratio effect before changing policy — supports defensible remediation decisions.

## 16. Glossary & appendix

### 16.1 Glossary

| Term | Meaning |
|---|---|
| **Atlas** | Orchestrator agent (Opus 4.8) — root coordinator that plans, routes, and delegates. |
| **Scribe** | Intake & JD Builder agent (Sonnet 4.6) — turns a manager's request into a structured, inclusive, leveled JD. |
| **Scout** | Sourcing agent (Sonnet 4.6, ParallelAgent) — finds candidates across channels via semantic search. |
| **Sift** | Screening agent (Haiku→Sonnet, SequentialAgent) — parses, extracts skills, scores, and ranks vs the JD. |
| **Gauge** | Assessment agent (Opus 4.8, LoopAgent) — generates/evaluates assessments against rubrics. |
| **Echo** | Engagement agent (Sonnet + Haiku) — candidate Q&A, outreach, nurture, anti-ghosting. |
| **Sync** | Interview Coordination agent (Haiku 4.5, A2A client) — schedules multi-party interviews. |
| **Aegis** | Compliance & Governance agent (Opus 4.8, guardrail) — bias/EEOC/EU-AI-Act guardrails, reasoning log, human-decision boundary. |
| **Verify** | Background & Reference agent (Haiku + Sonnet, ParallelAgent) — runs background and reference checks. |
| **Lens** | Analytics & Insights agent (Sonnet 4.6) — funnel metrics, source effectiveness, impact-ratio math. |
| **Bridge** | Onboarding Handoff agent (Haiku 4.5) — hands the hired candidate off to HRIS and IT provisioning. |
| **LangGraph** | Durable state-machine graph framework — nodes/edges/state, checkpointing, interrupts (HITL), streaming, time-travel. |
| **ADK** | Google Agent Development Kit — provides LLM, Workflow (Sequential/Parallel/Loop), and Custom agent types, plus AgentTools. |
| **A2A** | Agent2Agent protocol — cross-vendor agent interoperability via agent cards, tasks, messages, and artifacts. |
| **RAG** | Retrieval-Augmented Generation — grounding agent answers in retrieved documents (e.g., Echo over company_knowledge). |
| **SSE** | Server-Sent Events — one-way server→client stream for the mission-control feed (`agent.thinking`, `agent.token`, `stage.changed`, `approval.required`, `task.completed`). |
| **HITL** | Human-in-the-loop — a graph `interrupt()` that pauses for human approval (JD, shortlist, borderline, offer). |
| **Reasoning log** | Append-only record of every ranking/advancement/rejection decision — the EU AI Act explainability artifact. |
| **Time-travel / replay** | Reconstruction of any candidate's full decision path from saved checkpoints, in the Compliance & Audit view. |
| **Impact ratio** | Ratio of selection rates across protected groups, computed for the LL144 bias audit. |
| **AEDT** | Automated Employment Decision Tool — the regulated category (NYC LL144) that TalentMesh's scoring falls under. |
| **Four-fifths rule** | The 80% guideline for flagging disparate impact in selection rates (EEOC/LL144 context). |
| **Comp band** | A salary/equity/bonus range for a role family + level (e.g., BAND-ENG-L5). |
| **Screening threshold** | The per-req score cutoff (e.g., 0.72 for REQ-2026-0431) that drives advance/reject/human-review branching. |
| **Nurture loop** | Echo's LoopAgent flow that re-engages quiet or below-threshold candidates (e.g., Jordan Lee, 90-day re-engage) until response or cap. |
| **Agent Card** | An A2A capability-discovery document declaring a vendor agent's URL, skills, auth, and transport. |
| **Signed card** | An agent card with a cryptographic signature verified against its `signature_domain` before it is trusted. |
| **Version negotiation** | A2A handshake reconciling client/vendor protocol versions (e.g., client 1.0 → vendor 0.3 with a shim for ReferenceCheckAgent). |

### 16.2 Appendix — 11-agent roster

| Agent | Codename | Model | Primary job | ADK type |
|---|---|---|---|---|
| Orchestrator | Atlas | Opus 4.8 (`claude-opus-4-8`) | Root coordinator; plans, routes, delegates, handles fallback/A-B routing | LLM agent (root) |
| Intake & JD Builder | Scribe | Sonnet 4.6 (`claude-sonnet-4-6`) | Turns a manager's request into a structured, inclusive, leveled JD | LLM agent + artifacts |
| Sourcing | Scout | Sonnet 4.6 (`claude-sonnet-4-6`) | Finds candidates across channels via semantic search | Parallel workflow agent |
| Screening | Sift | Haiku 4.5 (parse) + Sonnet 4.6 (score) | Parses resumes, extracts skills, scores & ranks vs the JD | Sequential workflow agent |
| Assessment | Gauge | Opus 4.8 (`claude-opus-4-8`) | Generates/evaluates skills assessments against rubrics | Loop workflow agent |
| Engagement | Echo | Sonnet 4.6 (chat) + Haiku 4.5 (notifications) | Candidate Q&A, outreach, nurture, anti-ghosting | LLM agent + Loop (nurture) |
| Interview Coordination | Sync | Haiku 4.5 (`claude-haiku-4-5-20251001`) | Schedules multi-party interviews | LLM agent (A2A client) |
| Compliance & Governance | Aegis | Opus 4.8 (`claude-opus-4-8`) | Bias/EEOC/EU-AI-Act guardrails, reasoning logs, human-decision boundary | Custom + LLM agent |
| Background & Reference | Verify | Haiku 4.5 (orchestrate) + Sonnet 4.6 (summarize) | Runs background + reference checks | Parallel workflow agent (A2A client) |
| Analytics & Insights | Lens | Sonnet 4.6 (`claude-sonnet-4-6`) | Funnel metrics, source effectiveness, impact ratios | LLM agent + custom tools |
| Onboarding Handoff | Bridge | Haiku 4.5 (`claude-haiku-4-5-20251001`) | Hands the hired candidate off to HRIS + IT provisioning | LLM agent (A2A client) |


---

_End of PRD. Generated for TalentMesh v1.0 — recruiter experience._