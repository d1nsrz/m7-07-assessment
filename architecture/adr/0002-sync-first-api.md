# ADR 0002: Synchronous API as the Primary Serving Path

**Status:** Accepted  
**Date:** 2026-06-06

## Context

Home-screen recommendation loads are blocking — the user is staring at a spinner waiting for the content. This means we need a synchronous response. However we also need batch and async endpoints for pre-computation and bulk scoring jobs.

## Decision

The primary serving path is **synchronous** (`POST /v1/recommendations`). We also expose a batch endpoint (`POST /v1/recommendations/batch`) and an async job endpoint (`POST /v1/recommendations/jobs`) but these are secondary paths.

## Reasons

- **User experience requires sync** — A 120 ms p95 budget is tight but achievable sync. Making the app poll an async job for a home-screen load would add a lot of round trips and complexity for no benefit.
- **Batch endpoint serves pre-computation** — The nightly Airflow job uses the batch endpoint to pre-warm the Redis cache for active users. This way the batch logic goes through the same API contract and gets the same observability.
- **Async endpoint for A/B experiment scoring** — When the product team wants to score a new model candidate against the full active user base before promoting it, they use the async job endpoint. This decouples the long-running job from the sync path.

## Trade-offs

- **Sync path must be very reliable** — A latency spike on the sync endpoint directly hurts users. We mitigate this with strict timeouts (100 ms service timeout, with 20 ms left for network), circuit breakers, and the pre-computed fallback in Redis.
- **Three endpoint types to maintain** — More surface area in the API. We accept this because each type solves a clearly different use case.
