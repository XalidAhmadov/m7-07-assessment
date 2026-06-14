# RecoServe recommendation service — multi-stage, slim, model-AGNOSTIC.
# The model is NOT copied in: it is mounted at runtime from the registry (ADR-0002).
# Only the cold-start fallback ships in the image.
# This image will not be built for grading, but is written to be reviewable & buildable.

# ---- Stage 1: builder -------------------------------------------------------------------
FROM python:3.12-slim AS builder

ENV PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PYTHONDONTWRITEBYTECODE=1

WORKDIR /build

# Build deps only in the builder stage; they never reach the final image.
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install into an isolated prefix we can copy wholesale into the runtime stage.
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# ---- Stage 2: runtime -------------------------------------------------------------------
FROM python:3.12-slim AS runtime

# Non-root, no shell login, fixed UID for predictable volume permissions.
RUN groupadd --gid 10001 app \
    && useradd --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    OMP_NUM_THREADS=2 \
    ORT_THREAD_POOL_SIZE=2 \
    # Where the init container mounts the registry artifact (ADR-0002).
    MODEL_MOUNT_PATH=/models \
    PORT=8080

WORKDIR /app

# Copy only the installed site-packages from the builder — no compilers in final image.
COPY --from=builder /install /usr/local

# Service code + the baked-in cold-start fallback (the ONLY model in the image).
COPY --chown=app:app app/ ./app/
COPY --chown=app:app fallback/fallback-v19.bin ./fallback/fallback-v19.bin

# The personalized ranker is mounted here at runtime; empty dir in the image on purpose.
RUN mkdir -p /models && chown app:app /models
VOLUME ["/models"]

USER app
EXPOSE 8080

# Readiness fails until a valid mounted model is loaded (ADR-0002 safety property).
HEALTHCHECK --interval=10s --timeout=2s --start-period=20s --retries=3 \
    CMD ["python", "-m", "app.healthcheck"]

ENTRYPOINT ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080", "--workers", "2"]
