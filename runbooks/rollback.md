# Rollback Runbook — Product Recommender

**Owner:** On-call ML engineer  
**Last updated:** 2026-06-06

---

## When to Use This Runbook

Trigger a rollback when **any** of these alerts fire and don't resolve within 5 minutes:

| Alert | Threshold | Defined in |
|---|---|---|
| `HighErrorRate_Critical` | error rate > 2% for 5 min | `monitoring/alerts.yaml` |
| `HighP95Latency_Critical` | p95 > 200 ms for 5 min | `monitoring/alerts.yaml` |
| `ErrorBudgetBurnRate_1h` | burn rate > 14.4x in 1h | `monitoring/alerts.yaml` |
| `TooFewReplicas` | < 4 healthy replicas for 2 min | `monitoring/alerts.yaml` |

Do **not** rollback for `ModelVersionMismatch` alone — that alert fires during a deploy and is expected. If it's still active 30 minutes after deploy completes, investigate.

---

## Step 1 — Confirm the problem (< 2 min)

- [ ] Check Grafana dashboard: is error rate or p95 actually elevated, or is the alert a false positive?
- [ ] Check PagerDuty for context: was there a recent deploy? (check `#deploys` Slack channel)
- [ ] Identify the current and previous image tags:
  ```bash
  kubectl get deployment recommender -n recommender-prod \
    -o jsonpath='{.spec.template.spec.containers[0].image}'
  ```

---

## Step 2 — Roll back the Kubernetes deployment (< 5 min)

```bash
# Roll back to the previous revision
kubectl rollout undo deployment/recommender -n recommender-prod

# Watch the rollback complete
kubectl rollout status deployment/recommender -n recommender-prod --timeout=5m
```

If the previous revision is also broken (rare), specify the revision explicitly:

```bash
kubectl rollout history deployment/recommender -n recommender-prod
kubectl rollout undo deployment/recommender -n recommender-prod --to-revision=<N>
```

---

## Step 3 — Verify recovery (< 5 min)

- [ ] Error rate drops below 0.5% in Grafana within 3 minutes of rollback completing
- [ ] p95 latency drops below 120 ms within 3 minutes
- [ ] `ModelVersionMismatch` alert clears (all pods now report the same `X-Model-Version`)
- [ ] Run a quick manual check:
  ```bash
  curl -sf -X POST https://api.example.com/recommender/v1/recommendations \
    -H "Content-Type: application/json" \
    -H "X-Request-ID: rollback-verify-001" \
    -d '{"user_id": "test_user_001", "n": 5}' | jq .model_version
  ```
  The returned `model_version` should match the previous stable version.

---

## Step 4 — Communicate and document (< 10 min)

- [ ] Post in `#incidents` Slack: "Rolled back recommender to `<previous image tag>`. Monitoring for stability."
- [ ] Update the incident ticket with: what alert fired, what time rollback was done, what version was rolled back to
- [ ] Mark the failing model version as `archived` in MLflow (do **not** delete — keep for post-mortem)

---

## Step 5 — Post-mortem (within 24h)

- Identify root cause of the regression
- Check if offline evaluation metrics failed to catch it (if so, update thresholds in `lifecycle/model-registry.yaml`)
- Re-run the shadow test with the problematic model version to understand what was missed
- Decide whether to retrain or fix the training data before attempting another promotion
