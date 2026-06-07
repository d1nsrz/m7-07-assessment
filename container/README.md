# Container Image Plan

## Strategy: Bake Model Into Image

We bake the model weights into the image at build time (see ADR 0001). The weights are downloaded from S3 during the Docker build using a dedicated `model-fetcher` stage, so they never touch the application layer.

## Base Image

`python:3.11-slim` — Debian-based slim image. We chose this over Alpine because PyTorch has C extension dependencies that are annoying to compile on Alpine. The slim variant removes most of the unnecessary packages.

## Multi-Stage Build

| Stage | Purpose | What it adds |
|---|---|---|
| `model-fetcher` | Download weights from S3 | AWS CLI, model files |
| `builder` | Install Python deps | All packages in requirements.txt |
| `runtime` | Final image | Only app code + installed packages + model |

The final stage only copies what it actually needs from the previous stages. This keeps the image clean and avoids leaking build tools into runtime.

## Size Estimate

| Layer | Approx size |
|---|---|
| python:3.11-slim base | ~50 MB |
| Python dependencies (FastAPI, PyTorch CPU, numpy, redis-py) | ~210 MB |
| Model weights (model.pt + item_embeddings.npy) | ~120 MB |
| Application code | ~2 MB |
| **Total** | **~380 MB** |

PyTorch CPU-only wheel is the biggest single dependency (~170 MB). If size becomes a concern we can switch to ONNX Runtime (~30 MB) after converting the model to ONNX format — that would bring the image down to around 200 MB.

## Security Notes

- Container runs as non-root user (`appuser`, UID 1001)
- No SSH, no curl, no apt in the final image
- Trivy scan runs in CI before push (see `cicd/.github/workflows/deploy-model.yml`)
- Build arg `MODEL_VERSION` is pinned in CI — it matches the tag recorded in `lifecycle/model-registry.yaml`

## Image Tag Format

```
recommender:<git-sha>-<model-version>
# example: recommender:3f8a21b-v2.1.0
```

This is the same scheme the model registry expects (see `lifecycle/model-registry.yaml`, field `image_tag`).
