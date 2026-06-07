# MLOps Design Dossier — Scenario X: Personalized In-App Recommendations

## Executive Summary

This repo contains a full MLOps design for a personalized product recommendation system in a mobile retail app. The model scores items for each user when they open the home screen. We use a two-tower embedding model that handles both returning users (based on last 30 days history) and cold-start users who have no history yet. The system is designed to handle up to 800 requests per second with a p95 latency under 120 ms end-to-end. A/B testing support is baked in through model versioning and a feature flag layer.

## Architecture Diagram

See [architecture/architecture.md](architecture/architecture.md) for the full diagram.

```
Mobile App → API Gateway → Recommender Service → Feature Store
                                    ↓
                             Model (baked in image)
                                    ↓
                           Response + X-Model-Version header
```

## Key Numbers

| Metric | Value |
|---|---|
| Target peak RPS | 800 |
| p95 latency budget | 120 ms end-to-end |
| Inference budget (service only) | ≤ 70 ms |
| Availability SLO | 99.9 % |
| Error rate SLO | < 0.5 % |
| Model type | Two-tower embedding (~120 MB) |
| Model storage | Baked into container image |
| Replicas (steady state) | 6 pods (2 vCPU, 4 GB each) |
| Instance type | AWS c5.large |
| Estimated monthly cost | ~$520 |

## Navigation

| Area | Primary artifact |
|---|---|
| Architecture | [architecture/architecture.md](architecture/architecture.md) |
| Justification | [architecture/JUSTIFICATION.md](architecture/JUSTIFICATION.md) |
| ADR 0001 | [architecture/adr/0001-bake-model-into-image.md](architecture/adr/0001-bake-model-into-image.md) |
| ADR 0002 | [architecture/adr/0002-sync-first-api.md](architecture/adr/0002-sync-first-api.md) |
| Lifecycle | [lifecycle/lifecycle.md](lifecycle/lifecycle.md) |
| Model registry | [lifecycle/model-registry.yaml](lifecycle/model-registry.yaml) |
| Dockerfile | [container/Dockerfile](container/Dockerfile) |
| Container plan | [container/README.md](container/README.md) |
| API spec | [api/openapi.yaml](api/openapi.yaml) |
| Capacity plan | [serving/capacity-plan.md](serving/capacity-plan.md) |
| SLOs | [serving/slos.yaml](serving/slos.yaml) |
| Load test | [serving/load-test-plan.md](serving/load-test-plan.md) |
| CI/CD pipeline | [cicd/.github/workflows/deploy-model.yml](cicd/.github/workflows/deploy-model.yml) |
| Monitoring alerts | [monitoring/alerts.yaml](monitoring/alerts.yaml) |
| Rollback runbook | [runbooks/rollback.md](runbooks/rollback.md) |

## Open Questions

1. **Feature store latency** — the 120 ms budget assumes feature fetch takes ≤ 30 ms. We need to confirm with the data team that the online feature store can hit this at 800 RPS before we finalize the capacity plan.
2. **Cold-start fallback catalog** — right now cold-start users get a popularity-based fallback. Product team should confirm whether that is acceptable or if we need a separate onboarding model.
3. **A/B test routing** — we assumed the experiment layer lives in the API gateway. If product team wants the model itself to do multi-arm routing, the architecture changes a bit and we should discuss that early.
