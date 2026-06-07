# Load Test Plan

## Goals

Confirm that 6 pods of c5.large can serve 800 RPS at p95 ≤ 120 ms before the first production deploy, and find the saturation point.

## Tool

**k6** — runs in CI against the staging environment.

## Test Scenarios

### 1. Baseline (smoke test, runs in CI on every deploy)
- 10 RPS for 60 seconds
- Pass criterion: p95 < 120 ms, error rate = 0%

### 2. Ramp test (pre-production gate)
- Ramp from 0 to 800 RPS over 5 minutes, hold 800 RPS for 10 minutes, ramp down
- Pass criterion: p95 < 120 ms during the hold phase, error rate < 0.5%

### 3. Peak burst test
- Jump from 200 RPS to 800 RPS in 30 seconds (simulates app push notification causing a spike)
- Observe HPA scale-out time, check that p95 stays under 200 ms during scale-out
- No hard fail on latency during the burst window, but should recover to < 120 ms within 3 minutes

### 4. Saturation test (run manually, not in CI)
- Ramp to 1200 RPS (150% of peak)
- Find where the service starts returning errors
- Document the saturation RPS so we know the safety margin

## k6 Script Skeleton

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

const BASE_URL = "https://staging-api.example.com/recommender";

export const options = {
  scenarios: {
    ramp: {
      executor: "ramping-arrival-rate",
      startRate: 0,
      timeUnit: "1s",
      preAllocatedVUs: 200,
      stages: [
        { target: 800, duration: "5m" },
        { target: 800, duration: "10m" },
        { target: 0,   duration: "2m" },
      ],
    },
  },
  thresholds: {
    "http_req_duration{scenario:ramp}": ["p(95)<120"],
    "http_req_failed{scenario:ramp}":   ["rate<0.005"],
  },
};

const USER_IDS = open("./test_user_ids.json");  // 10K sample user IDs

export default function () {
  const userId = USER_IDS[Math.floor(Math.random() * USER_IDS.length)];
  const payload = JSON.stringify({ user_id: userId, n: 10 });
  const params = {
    headers: {
      "Content-Type": "application/json",
      "X-Request-ID": `test-${__VU}-${__ITER}`,
    },
  };

  const res = http.post(`${BASE_URL}/v1/recommendations`, payload, params);
  check(res, {
    "status 200": (r) => r.status === 200,
    "has model version header": (r) => r.headers["X-Model-Version"] !== undefined,
    "p95 under budget": (r) => r.timings.duration < 120,
  });
}
```

## Success Criteria Summary

| Test | RPS | p95 threshold | Error threshold |
|---|---|---|---|
| Smoke | 10 | 120 ms | 0% |
| Ramp | 800 | 120 ms | 0.5% |
| Burst recovery | 800 | 200 ms (burst), 120 ms (steady) | 1% during burst |

## Notes

- Tests run against staging, not production
- Test user IDs must be real IDs that exist in the staging feature store
- Cold-start users (~20% of traffic) should be included in the test data to make sure fallback doesn't cause latency spikes
