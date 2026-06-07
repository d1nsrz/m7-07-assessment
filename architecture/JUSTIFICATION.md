# Architecture Justification

## Pattern Choice: Synchronous Inference with Pre-Computed Embeddings

We chose a **real-time synchronous inference** pattern with **pre-computed item embeddings** rather than full online inference or pure batch pre-computation.

### Why not fully online inference?

Full online inference (scoring all items at request time) would take too long. With ~50K items in the catalog, even a fast dot-product would exceed the 120 ms budget unless we have very expensive hardware. Pre-computing item embeddings offline and doing a fast ANN (approximate nearest neighbor) lookup at request time is the standard approach here.

### Why not fully batch/pre-computed recommendations?

Pure pre-computation (store top-K for every user in Redis, serve directly) would work but it means recommendations are stale by hours. The product team specifically wants to use real-time browsing signals — if a user just looked at running shoes, the next home screen load should reflect that. So we need at least partial real-time feature lookup.

### Why two-tower model?

Two-tower is a well established architecture for retrieval at scale. User tower and item tower are trained jointly but at serve time the item embeddings can be pre-computed. This gives us the best of both worlds: personalization from real-time user features, fast retrieval from pre-computed item embeddings.

### Trade-offs accepted

| Decision | Trade-off |
|---|---|
| Bake model into image | Larger images (~400 MB), but simpler ops and faster cold start |
| Redis for feature store | Adds ~30 ms per request but keeps service stateless |
| Stateless pods | Can scale horizontally easily; no sticky sessions needed |
| Pre-compute item embeddings | Slight staleness (~1h) when catalog changes; acceptable for retail |

### What we'd revisit at scale

If the catalog grows above ~500K items, ANN lookup might need a dedicated vector DB (like Pinecone or Weaviate) instead of Redis. At current scale Redis is fine and simpler.
