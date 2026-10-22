# Runbook 02: Cost spike

**Alerts:**
- `AgentCostSpike`: hourly spend > 2× the same hour last week, sustained for 30 minutes.
- `AgentCostPerResolutionHigh`: > $0.12 p50 for 1 hour.
- Global and tenant budget alerts at 80% and 100%.

**Severity:** Sev2. Raise to Sev1 if the global daily budget is forecast to run out more than 4 hours before end of day.
**Owner:** Platform SRE triages. Agent Eng owns the fix if the cause is a behaviour or prompt change.

## Symptoms
Spend climbs faster than traffic. Note that a traffic spike with flat cost per session is **not** a cost incident. That is a capacity or budget-planning question.

## Triage

Break spend down as **sessions × steps/session × tokens/step × $/token**, and find which factor moved.

```promql
# 1. Is it traffic? Compare sessions vs spend growth
sum(rate(agent_sessions_total[1h])) / sum(rate(agent_sessions_total[1h] offset 1w))
sum(rate(agent_cost_usd_total[1h]))  / sum(rate(agent_cost_usd_total[1h] offset 1w))

# 2. Which model / release / tenant tier?
topk(5, sum by (model, release)(rate(agent_cost_usd_total[1h])))
topk(5, sum by (tenant_tier)(rate(agent_cost_usd_total[1h])))

# 3. Steps and tokens
histogram_quantile(0.95, sum by (le)(rate(agent_steps_per_session_bucket[1h])))
sum by (token_type)(rate(gen_ai_client_token_usage_sum[1h]))

# 4. Cache hit ratio collapsed? (a prompt change can break prefix caching)
sum(rate(gen_ai_client_token_usage_sum{token_type="cache_read"}[1h]))
  / sum(rate(gen_ai_client_token_usage_sum{token_type="input"}[1h]))
```

Common root causes, roughly in order of how often they happen:

1. **The router is sending more traffic to the large model.** A router config change or classifier drift caused it.
2. **Prompt caching broke.** Someone added a dynamic value (a timestamp or a user name) to the system prompt prefix.
3. **Context growth.** History stops being summarised, or retrieval returns more chunks (`top_k` changed).
4. **Retries or loops.** Provider errors cause retries, or the agent repeats tool calls. If `agent_budget_exhausted_total{limit="loop"}` is rising, go to runbook 04.
5. **A single tenant or bot.** Scripted traffic, or abuse, through one integration.
6. **A provider price change, or a model alias moving** to a more expensive version.

## Mitigation

| Cause | Action |
|---|---|
| A new release | Roll it back |
| Router skew | Pin the router to the previous config, or set `agent.router.force_model=fallback` for intents marked `low_complexity` |
| Cache broken | Roll back the prompt bundle. Do **not** hot-patch the prompt outside the release process |
| A single tenant or abusive traffic | Apply a per-tenant rate limit at the gateway. If the tenant is at 100% of budget, it auto-routes to the small model. Inform the account owner |
| Global budget at 100% | `agent.degraded_mode=true` turns on automatically. Tell Support Ops, because the escalation rate will rise |
| Session spend hitting the cap a lot | Confirm the per-session $0.40 cap is enforced (`agent_budget_exhausted_total{limit="usd"}`). If it is not firing, treat it as a bug and lower `max_steps` |

## Verify
- Cost per resolved session is back within 10% of baseline.
- The cache hit ratio has recovered.
- Task success has not dropped because of the mitigation. Degraded mode trades quality for spend, so record that trade-off in the incident.

## Follow-ups
- Calculate the extra spend: run the cost-ledger query for the incident window minus the baseline. Put the number in the post-mortem.
- Add a cache-hit-ratio check to the shadow canary gate if it was not caught there.
