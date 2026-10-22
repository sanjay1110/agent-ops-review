# Runbook 01: Latency regression

**Alerts:** `AgentTTFTSLOBurn` (p95 TTFT > 2.5 s, multi-window burn), `AgentSessionLatencySLOBurn` (p95 resolved-session duration > 45 s)
**Severity:** Sev2 by default. Raise to Sev1 if availability is also burning, or if p95 TTFT exceeds 10 s for more than 10 minutes.
**Owner:** Platform SRE. Pull in Agent Eng if the cause turns out to be steps or tokens.

## Symptoms
Customers wait before the first token, or sessions take much longer to resolve. Abandonment rises, because `outcome="abandoned"` goes up.

## Triage (first 10 minutes)

Session latency is roughly *steps × (LLM time + tool time)*. Find out which of the three terms moved.

| Check | Query | If it moved → likely cause |
|---|---|---|
| Is it one release? | `histogram_quantile(0.95, sum by (le, release)(rate(agent_ttft_seconds_bucket[10m])))` | Canary or new release → **roll back** |
| LLM call latency by model and provider | `histogram_quantile(0.95, sum by (le, model)(rate(gen_ai_client_operation_duration_seconds_bucket[10m])))` | Provider degradation, rate limiting, or a larger model being routed |
| Provider errors and 429s | `sum by (error_type)(rate(gen_ai_client_operation_duration_seconds_count{error_type!=""}[5m]))` | Retries are inflating latency |
| Tool latency | `topk(5, histogram_quantile(0.95, sum by (le, tool)(rate(agent_tool_duration_seconds_bucket[10m]))))` | Downstream dependency (CRM, KB, billing) |
| Steps per session | `histogram_quantile(0.95, sum by (le)(rate(agent_steps_per_session_bucket[30m])))` | Behaviour change: more planning or replanning (see runbook 03) |
| Output tokens per call | `gen_ai_client_token_usage{token_type="output"}` p95 | The prompt or model is producing longer outputs |

TraceQL to look at real examples:
```traceql
{ name = "agent.session" && duration > 45s } >> { duration > 5s }
```

## Mitigation

| Cause | Action |
|---|---|
| A new release | `releasectl rollback resolve --to <prev> --reason INC-…` |
| Provider slow, or returning 429/5xx | `agent.router.force_model=fallback`, or fail over to the secondary provider region. Check the provider status page. Lower the request concurrency so you do not make 429 storms worse |
| A slow tool | Lower that tool's timeout (default 8 s → 4 s) and turn on its circuit breaker. For the KB, serve from the cached top answers |
| Steps per session went up | Lower the step cap temporarily (12 → 8) using `agent.limits.max_steps`. Investigate further with runbook 03 |
| Capacity, orchestrator CPU or queue saturation | Scale out the orchestrator. Check the HPA and the connection pool to the provider |

## Verify
- p95 TTFT is back under the SLO for 30 minutes.
- The abandonment rate is back to baseline.
- Fallback routing has not dropped task success by more than 2 points. Check the eval dashboard. The fallback model is cheaper but weaker.

## Follow-ups
- If the cause was a provider: record it in the provider SLA log, and look again at the multi-provider routing priority.
- If the cause was a release: add a latency regression case to the shadow-stage gate.
