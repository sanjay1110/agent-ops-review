# ADR 0001: Production operations for the Resolve agent

- **Status:** Proposed
- **Date:** 2026-10-22
- **Deciders:** Platform SRE, Agent Engineering, Support Ops

## Context

Resolve is non-deterministic, runs a variable number of steps per session, pays per token, and can take irreversible actions (refunds). Traditional request/response monitoring misses its main failure modes:
- silent quality decay
- runaway loops
- cost spikes from verbose prompts or retries
- tool misuse

We need one operating model that covers reliability, quality, cost and ownership.

## Decision

### 1. Observability strategy
- **OpenTelemetry everywhere**, using the GenAI semantic conventions (`gen_ai.*`) for model calls. One root span per session, with child spans for each step, LLM call, tool call, retrieval and guardrail check.
- **Three stores, one correlation key.** Metrics, traces and logs all carry `session.id`, `agent.release` and a hashed `tenant.id`.
- **Prompts and completions never go on spans.** They go to a redacted transcript store (30-day retention). Spans hold a pointer to the transcript.
- **Tail-based sampling.** Keep 100% of sessions that error, escalate, exhaust a budget, trip a guardrail or are slow (>p99). Keep 10% of all other sessions.
- **Online evaluation.** An LLM judge plus deterministic rules score a 5% sample of completed traces, and 100% of traces that issued refunds. Results become metrics and warehouse rows.

### 2. SLIs and SLOs (28-day rolling window)

| SLI | Definition | SLO |
|---|---|---|
| Availability | Sessions not ending in platform error ÷ all sessions | **99.5%** |
| Responsiveness | Time to first token for agent reply, p95 | **≤ 2.5 s** |
| Completion latency | Session duration for resolved sessions, p95 | **≤ 45 s** |
| Task success | Sampled sessions judged "correctly resolved or correctly escalated" | **≥ 88%** |
| Tool reliability | Tool calls that succeed (excludes valid business rejections) | **≥ 99.0%** |
| Action safety | Refunds failing policy check in post-hoc audit | **≤ 0.05%** (paging, zero-tolerance trend) |

**Error budget policy:**
- At 50% of budget burned: freeze non-fix releases for that SLO's owner.
- At 100% burned: all releases go through a full canary at 1% for 24 h, and a reliability item takes priority in the next sprint.

### 3. Cost controls

| Layer | Control | On breach |
|---|---|---|
| Step | `max_output_tokens` per call. Context window trimmed to a 24k-token budget | Truncate, summarise history |
| Session | ≤ 12 agent steps, ≤ 20 tool calls, ≤ $0.40 spend. Loop detector stops after 3 identical tool calls | Hand off to human, emit `agent_budget_exhausted_total` |
| Tenant | Daily spend budget per tenant | 80%: alert account owner. 100%: route to small model |
| Global | Daily and monthly spend budget | 50/80/100% alerts. At 100%: "degraded mode" flag (small model, retrieval-only answers) |
| Structural | Model routing (small model for classify/extract), prompt caching, retry caps (max 2, jittered) | Reviewed weekly in cost report |

Unit-economics target: **cost per resolved session ≤ $0.12 (p50)**. This is tracked per release and is a canary gate, not an SLO.

### 4. On-call responsibilities

| Rotation | Owns | Pages for |
|---|---|---|
| **Primary: Platform SRE** | Gateway, orchestrator runtime, collector, data stores, provider connectivity | Availability, latency, tool reliability, cost spike |
| **Secondary: Agent Engineering** | Prompts, router, tools, guardrails, evals, releases | Task success, drift, loops, action safety |
| **Business contact: Support Ops lead** (business hours) | Handoff queue capacity, customer comms | Sev1/Sev2 with customer impact |

- **Sev1:** customer-visible outage, or unsafe/irreversible actions at scale. Page within 5 minutes. Both rotations engaged.
- **Sev2:** SLO burn rate >6×, or a cost spike over 2× baseline.
- **Sev3:** ticket only, next business day.

Primary acks within 15 minutes. Either rotation may flip the **kill switches** without approval:
- `agent.degraded_mode`
- `agent.refunds_enabled=false`
- roll back `agent_release`

LLM provider issues escalate through the provider's status page and support contract. The contact details live in the private on-call wiki, not in this repo.

## Consequences
- ➕ Quality and cost regressions page someone, instead of surfacing in a monthly review.
- ➕ The release bundle gives every incident a single rollback lever.
- ➖ Online evaluation adds ~3% to model spend. The LLM judge itself needs calibration against human labels each quarter.
- ➖ Tail sampling means non-sampled healthy traces are unavailable for ad-hoc debugging. Metrics remain 100% complete.

## Alternatives considered
- **Head-based 100% tracing:** rejected on storage cost and the PII surface.
- **Vendor-only LLM observability SaaS:** rejected as the system of record because of lock-in. It can still consume OTLP as a secondary exporter.
- **Quality reviewed manually each week:** rejected because detection is too slow for drift.
