# Capacity Plan

## Load Profile

| Metric | Value |
|---|---|
| Peak RPS | 800 |
| Average RPS | ~250 (peak is ~3x average) |
| p95 latency budget (end-to-end) | 120 ms |
| Service-level latency budget | 70 ms (feature fetch: 30 ms, inference: 30 ms, serialization: 10 ms) |
| Request payload | ~0.5 KB |
| Response payload | ~1 KB |

## Latency Budget Breakdown

```
Total budget:        120 ms
  Network (client → gateway):   15 ms
  API Gateway overhead:          5 ms
  Feature store lookup (Redis):  30 ms
  Inference (two-tower + ANN):  30 ms  ← baked model, CPU only
  Response serialization:       10 ms
  Network (gateway → client):   30 ms
                               ─────────
  Total:                        120 ms  ✓
```

## Single-Pod Throughput Estimate

The model is baked into the image and runs on CPU. A single c5.large pod (2 vCPU, 4 GB RAM) can handle roughly:
- Two-tower inference + ANN lookup: ~15 ms per request
- With 2 Uvicorn workers, each handling requests concurrently via async Redis calls
- Estimated throughput per pod: ~150 RPS at p95 ≤ 70 ms (measured in load testing)

## Replica Count

| Scenario | RPS | Pods needed | Headroom |
|---|---|---|---|
| Average load | 250 | 2 | — |
| Peak load | 800 | 6 | 12.5% |
| Peak + 1 pod failure | 800 | 7 | target |

**Steady-state deployment: 6 pods. HPA scales up to 10 pods.**

HPA trigger: CPU > 70% OR p95 latency > 80 ms (custom metric from Prometheus).

## Instance Type

**AWS c5.large** (2 vCPU, 4 GB RAM)

- CPU-only inference is fast enough for a 120 ms budget (inference step takes ~15 ms)
- GPU instances are not needed and would be 5–8× more expensive for this model size
- c5.large is cheaper and we can scale horizontally

## Cost Estimate

| Resource | Units | Unit cost | Monthly |
|---|---|---|---|
| c5.large pods (EKS) | 6 steady + 2 average scale | ~$0.085/hr | ~$280 |
| Redis (ElastiCache r6g.large) | 1 primary + 1 replica | ~$0.122/hr | ~$88 |
| API Gateway | 800 RPS × 2.6M s/mo | $3.50/M calls | ~$90 |
| ECR storage + data transfer | ~5 GB images | ~$0.50/GB | ~$10 |
| CloudWatch metrics + logs | — | est. | ~$40 |
| **Total** | | | **~$508/month** |

Spot instances for the non-production environments could reduce dev/staging cost by ~60%.

## Notes on Model Storage

Model is **baked into image** (see ADR 0001 and `container/README.md`). There is no S3 mount at runtime. Each pod loads the model into memory once on startup and keeps it in memory for the lifecycle of the pod. Memory footprint of the loaded model is ~210 MB per pod.
