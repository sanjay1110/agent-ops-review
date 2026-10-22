# Runbook 04: Runaway loop or unsafe action

**Alerts:**
- `AgentBudgetExhaustionSpike`: `agent_budget_exhausted_total` rate is more than 3× baseline.
- `AgentRefundPolicyViolation`: any refund fails the post-hoc policy audit.
- `AgentRefundVolumeAnomaly`: refund $ per hour is more than 2× the same hour last week.

**Severity:** a loop is Sev2. **Any unsafe irreversible action is Sev1.**
**Owner:** Agent Engineering. Support Ops and Finance join for Sev1.

## Immediate actions for an unsafe action (do these first, then think)

1. `agent.refunds_enabled=false`. The agent can then only propose refunds, and a human approves them.
2. Roll back if a release changed in the last 72 hours.
3. Freeze evidence: set `retain=true` on transcripts for every session with `side_effect="irreversible"` in the window.

## Triage

### Loops
```traceql
{ name = "agent.outcome" && span.outcome = "budget_exhausted" }
```
- Group by `limit`. If it is `loop`, the same tool is being called with the same `tool.args_hash` again and again. If it is `steps`, the agent keeps re-planning without making progress.
- Common causes:
  - A tool returns an error the agent cannot understand. A schema change or a new 4xx body format often causes this.
  - A guardrail rewrites the output, and the agent keeps retrying.
  - A model drift produces malformed tool arguments (`schema_invalid`).
- Check `agent_tool_calls_total{status="error"}` by tool, and whether the error started at a deploy of the *tool's* service. That service may belong to another team.

### Unsafe actions
List every irreversible action in the window and reconcile it against policy:
```sql
SELECT r.refund_id, r.session_id, r.amount_usd, r.policy_rule, r.release,
       a.audit_result, a.audit_reason
FROM refunds r JOIN refund_audit a USING (refund_id)
WHERE r.created_at BETWEEN :start AND :end
  AND (a.audit_result = 'fail' OR r.amount_usd > :cap)
ORDER BY r.created_at;
```
Find out which of these happened:
- **Policy gap.** The policy allowed something it should not have.
- **Prompt injection or manipulation.** The customer talked the agent into it. Look for a guardrail `allow` on a manipulative input.
- **Wrong tool arguments.** The wrong order id, or an amount in cents vs dollars.
- **Bypassed control.** The cap or idempotency check failed.

## Mitigation

| Cause | Action |
|---|---|
| Tool error loop | Fix or roll back the tool. Meanwhile, disable that tool in the router (`tools.disabled += [x]`). The agent then hands off for those intents |
| Re-planning loop after a release | Roll back the release. Lower `max_steps` temporarily |
| Injection or manipulation | Keep refunds off. Add the pattern to the input guardrail. Review similar sessions in the last 7 days |
| Control bypass (cap or idempotency) | Refunds stay off until the fix is deployed **and** tested against replayed sessions |

## Remediation of side effects
- Finance and Support Ops decide on each incorrect refund: reverse it, or keep it as a goodwill gesture. The agent team **does not** reverse charges itself.
- Customer communication goes through Support Ops templates.
- Record the total $ impact and the number of affected customers for the post-mortem.

## Verify
- The loop and budget-exhaustion rate is back to baseline.
- 24 hours of refunds proposed in human-approval mode all pass the audit, before you turn `refunds_enabled` back on (a staged re-enable, at the 5% canary first).
