# ── Stage 1: download model weights ──────────────────────────────────────────
FROM python:3.11-slim AS model-fetcher

ARG MODEL_VERSION=v2.1.0
ARG S3_BUCKET=mlops-model-artifacts

RUN pip install --no-cache-dir awscli

# Download model weights from S3 at build time (bake-in strategy)
RUN aws s3 cp s3://${S3_BUCKET}/recommender/${MODEL_VERSION}/model.pt /model/model.pt \
    && aws s3 cp s3://${S3_BUCKET}/recommender/${MODEL_VERSION}/item_embeddings.npy /model/item_embeddings.npy


# ── Stage 2: build dependencies ───────────────────────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /build
COPY requirements.txt .

RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


# ── Stage 3: final runtime image ──────────────────────────────────────────────
FROM python:3.11-slim AS runtime

# non-root user for security
RUN useradd -m -u 1001 appuser

WORKDIR /app

# copy installed packages from builder
COPY --from=builder /install /usr/local

# copy model weights from fetcher
COPY --from=model-fetcher /model /app/model

# copy application code
COPY --chown=appuser:appuser src/ /app/src/

USER appuser

ENV MODEL_PATH=/app/model/model.pt
ENV EMBEDDINGS_PATH=/app/model/item_embeddings.npy
ENV PORT=8080

EXPOSE 8080

HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')"

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080", "--workers", "2"]
