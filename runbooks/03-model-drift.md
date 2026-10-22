# Runbook 03: Model / quality drift

**Alerts:**
- `AgentTaskSuccessSLOBurn`: online-eval task success is burning its 88% SLO budget.
- `AgentEscalationRateHigh`: more than 1.25× the 7-day baseline.
- `AgentInputDrift`: input-embedding drift score is above its threshold.
- `AgentRetrievalScoreDrop`: p50 `retrieval_top_score` is down more than 15%.

**Severity:** Sev2. Raise to Sev1 if wrong answers involve money, policy or safety. Examples: wrong refund eligibility, or giving out incorrect legal or billing terms.
**Owner:** Agent Engineering.

## What "drift" means here

Drift comes in four kinds. Each has a different fix, so the first job is to find out which kind you have.

| Type | What changed | Typical signal |
|---|---|---|
| **Model drift** | The provider updated the model, or an alias now points somewhere else | Quality shift with *no* release on our side. Output length or format changes. More `schema_invalid` tool calls |
| **Data / knowledge drift** | The KB index was rebuilt, or documents are stale | `retrieval_top_score` drops. The groundedness eval drops. Failures cluster on certain topics |
| **Input drift** | Customers ask about new things: a launch, an outage, a new policy | `agent_input_embedding_drift` rises. New intent clusters appear. Escalations rise for one intent |
| **Evaluator drift** | The LLM judge changed or lost calibration | Scores move but business metrics (resolution rate, CSAT) do not. `agent_eval_judge_agreement` drops |

## Triage

1. **Does a business metric agree?** Check the resolution rate, thumbs-down rate and reopen rate. If only the judge score moved, suspect evaluator drift. Check judge agreement and the judge model version **before** you change the agent.
2. **Did anything change?**
   - `releasectl history resolve --since 7d`
   - The KB index version on recent spans (`kb.index_version`)
   - Provider changelogs. Confirm the model version on spans: `gen_ai.response.model` should match the pinned version.
3. **Where are the failures concentrated?** Slice eval failures by intent, channel and language in the warehouse:
   ```sql
   SELECT intent, COUNT(*) n, AVG(eval_task_success) success,
          AVG(eval_grounded) grounded, AVG(retrieval_top_score) top_score
   FROM agent_sessions
   WHERE ended_at > NOW() - INTERVAL '24 hours'
   GROUP BY intent HAVING COUNT(*) > 50
   ORDER BY success ASC LIMIT 10;
   ```
4. **Read 20 failing transcripts.** Do this before you form a theory. Pull trace ids from `{ span.eval.task_success < 0.5 }` and open their redacted transcripts. Tag each one with a failure category: wrong tool, wrong argument, hallucinated policy, stale fact, refused when it should not have, or format error.

## Mitigation

| Type | Action |
|---|---|
| Model drift | Pin to the last known-good dated model version through a new release bundle, or roll back. If the provider removed that version, route the affected intents to the fallback model and start an expedited eval of the replacement |
| Data drift | Roll back to the previous KB index (`kb_index` in the bundle). Fix the ingestion job. Re-index |
| Input drift (new topic) | Add a guardrail or a router rule that sends the new intent to a human (`handoff_intents += [...]`). Tell Support Ops. Write a KB article. Add eval cases |
| Evaluator drift | Freeze the judge to its previous version. Re-run calibration against 200 human-labelled sessions. While the judge is suspect, **suspend automatic canary gates that depend on it**, and promote releases only manually |

## Verify
- Task success and the business metrics have both recovered for 24 hours, across a weekday peak.
- The failing intent cluster is back to baseline, or is being handed off on purpose.

## Follow-ups
- Add the 20 tagged transcripts, redacted, as regression cases in the offline eval suite.
- If the provider changed behaviour behind a pinned version: escalate to the provider, and record it in the vendor risk log.
