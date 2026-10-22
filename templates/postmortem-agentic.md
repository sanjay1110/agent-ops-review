# Post-mortem: <INC-#### short title>

> Blameless. We are looking for the conditions that allowed a failure, not for a person or a model to blame. "The model hallucinated" is never a root cause. It is a symptom. Ask why the system let that output through and act on it.

| Field | Value |
|---|---|
| Severity / duration | Sev_ · detected __:__ → mitigated __:__ → resolved __:__ (timezone) |
| Incident commander / author | |
| `agent_release` (bad → good) | `r____` → `r____` |
| Bundle component that changed | model ☐ prompt ☐ tools/schema ☐ router ☐ guardrails ☐ KB index ☐ provider-side ☐ none ☐ |
| Detected by | SLO alert ☐ canary gate ☐ eval ☐ customer/support ☐ finance/audit ☐ engineer ☐ |

## 1. Summary
Two or three sentences: what the agent did wrong, who was affected, and how it was stopped.

## 2. Impact
| Dimension | Amount |
|---|---|
| Sessions affected / % of traffic | |
| Wrong or harmful answers (estimated from eval sample) | |
| **Irreversible actions taken** (count, $, reversed?) | |
| Extra model spend vs baseline | $ |
| Escalations to humans / extra handle time | |
| SLO error budget consumed | availability _% · task success _% · latency _% |
| Data or privacy exposure | none ☐ / describe |

## 3. Timeline
| Time | Event (cite trace id / dashboard / release event) |
|---|---|
| | Change shipped / provider changed |
| | First bad trajectory (find it in traces, after the fact) |
| | Alert fired / first report |
| | Mitigation lever pulled (which one) |
| | Recovered |

**Detection gap:** the time from the first bad trajectory to detection = ___. Why was it this long?

## 4. Trajectory analysis (agent-specific)
Reconstruct at least one representative failing trajectory from the trace and the redacted transcript.

| Step | Agent intent | Action (model / tool + args summary) | Observation returned | Where it went wrong |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |

Classify the failure. Tick all that apply:
- ☐ **Perception:** misread the user, the retrieval or the tool output
- ☐ **Planning:** wrong goal or decomposition, or kept re-planning without progress
- ☐ **Tool use:** wrong tool, wrong arguments, schema mismatch, misread a tool error
- ☐ **Grounding:** stated something not supported by the retrieved context, or the context was stale
- ☐ **Control:** a loop, step or budget limit, guardrail, cap or idempotency check failed or was missing
- ☐ **Adversarial:** prompt injection or social engineering, from the user or from retrieved content
- ☐ **Non-determinism:** reproduces only some of the time. Record the reproduction rate over N replays
- ☐ **Evaluation:** the judge or eval suite missed it, or itself drifted
- ☐ **Dependency:** a provider, a downstream tool service, or the KB pipeline

**Reproduction:** replayed __ times with the same inputs on the bad release → failed __/__. On the good release → failed __/__.

## 5. Contributing factors
"Why did the system allow this?" Work through each layer:
- **Release process:** did the offline eval, shadow run or canary gates cover this case? If not, why not?
- **Guardrails and limits:** which control should have caught it?
- **Observability:** what signal existed but did not alert, and what signal was missing?
- **Human factors and process:** ownership, handoffs, documentation.

## 6. What went well / where we got lucky

## 7. Action items
Each item needs an owner and a due date. At least one item must be in each of the first three rows.

| Type | Action | Owner | Due | Ticket |
|---|---|---|---|---|
| **Eval** (regression case added) | | | | |
| **Detection** (alert / gate / SLI) | | | | |
| **Prevention** (guardrail, limit, schema, policy) | | | | |
| Side-effect remediation (reversals, customer comms) | | | | |
| Process / docs / runbook update | | | | |

## 8. Evidence
Trace ids, dashboard snapshots, query links, the release diff. **Do not paste customer data or raw transcripts.** Link to the retained, access-controlled transcripts by reference instead.
