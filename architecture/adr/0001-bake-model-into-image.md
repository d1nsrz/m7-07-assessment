# ADR 0001: Bake Model Weights Into the Container Image

**Status:** Accepted  
**Date:** 2026-06-06

## Context

We need to decide how the inference service gets access to the model weights (~120 MB). Two main options: bake the weights into the Docker image at build time, or mount them at runtime from an S3/EFS volume.

## Decision

We will **bake the model weights into the container image**.

The image will be tagged as `recommender:<git-sha>-<model-version>`, for example `recommender:3f8a21b-v2.1.0`. This tag scheme is what the model registry expects when it records a deployment.

## Reasons

- **Startup time** — A pod that mounts weights from S3 has to download ~120 MB before it can serve traffic. With baked weights, the image is pulled once to the node and pods start in seconds, not minutes. At 800 RPS we can't afford slow scale-out.
- **Immutability** — Every image tag points to exactly one model version. There is no way to accidentally run the wrong weights in production.
- **Rollback simplicity** — Rolling back means pointing the deployment to the previous image tag. No need to also rollback a separate volume or S3 object.

## Trade-offs

- **Image size** — The final image will be around 380–420 MB. ECR pull on a new node takes ~30 seconds on a warm cache, ~90 seconds cold. This is acceptable given that scale-out events are not frequent.
- **Separate CI for model updates** — Every time the ML team promotes a new model, a new image must be built and pushed, even if the code didn't change. We mitigate this with a dedicated `build-model-image` job in the CI pipeline.

## Rejected alternative: Mount at runtime

Mounting from S3 was rejected because it adds operational complexity (IAM roles, S3 lifecycle, potential download failures) and makes startup slower. For a 120 MB model the benefits of separate storage don't justify the added risk.
