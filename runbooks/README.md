# Incident-response runbooks

| # | Scenario | Primary alert(s) | First owner |
|---|---|---|---|
| 01 | [Latency regression](01-latency-regression.md) | `AgentTTFTSLOBurn`, `AgentSessionLatencySLOBurn` | Platform SRE |
| 02 | [Cost spike](02-cost-spike.md) | `AgentCostSpike`, `AgentCostPerResolutionHigh`, budget 80/100% | Platform SRE → Agent Eng |
| 03 | [Model / quality drift](03-model-drift.md) | `AgentTaskSuccessSLOBurn`, `AgentInputDrift`, `AgentEscalationRateHigh` | Agent Eng |
| 04 | [Runaway loop or unsafe action](04-runaway-loop-unsafe-action.md) | `AgentBudgetExhaustionSpike`, `AgentRefundPolicyViolation` | Agent Eng (Sev1 if refunds) |

## Applies to every incident

1. **Ack, then declare.** Open an `INC-` channel, name an incident commander (IC), and set a severity using the ADR rubric.
2. **Ask the two agentic questions first.**
   - *Did a release change?* Run `releasectl history resolve --since 24h` and check provider status pages. Look for model or KB index changes.
   - *Is the agent taking actions in the world that we need to stop?* Look at the refund rate and the guardrail blocks.
3. **Mitigate before you diagnose.** These levers are listed from least to most disruptive:

| Lever | Effect | Who can pull |
|---|---|---|
| `releasectl rollback` | Return to the previous bundle | Either rotation |
| `agent.router.force_model=fallback` | Route all traffic to the small/fallback model | Either rotation |
| `agent.refunds_enabled=false` | The agent can only *offer* refunds. A human approves them | Either rotation |
| `agent.degraded_mode=true` | Retrieval-only answers, small model, tighter step cap | Either rotation |
| `agent.handoff_all=true` | Every session goes to the human queue. Tell Support Ops first | IC |

4. **Preserve evidence.** Pin example trace ids. Add `retain=true` to the affected sessions' transcripts so they survive the 30-day retention window.
5. **Update stakeholders** every 30 minutes (Sev1) or every 60 minutes (Sev2).
6. **Close out.** Write a post-mortem using [`templates/postmortem-agentic.md`](../templates/postmortem-agentic.md) for every Sev1/Sev2, and for any incident involving an irreversible action.
