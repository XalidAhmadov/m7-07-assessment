# Container plan

## What's in the image

A model-agnostic inference service. It's the FastAPI app, ONNX Runtime (CPU build), and the
cold-start fallback model — and that's deliberately it. The real personalized ranker isn't in
here; it gets mounted at runtime from the registry. That's ADR-0002, and it's the thing that lets
product flip experiment arms without us rebuilding anything.

## Bake or mount?

I went back and forth on this, so here's the actual reasoning rather than just the answer.

If you bake the model in, provenance is dead simple (the image *is* the version) but every new
experiment arm or model refresh becomes a full build → scan → rolling deploy. With continuous A/B
that's a deploy several times a day just to change which model is live. Not workable.

If you mount it, a new arm is a config change and a pod restart. One image covers every arm, so
there's less to build and less surface to keep patched. The cost is that startup now depends on
the registry being reachable and the artifact being intact.

So: **mount the ranker, bake the fallback.** The fallback (`fallback-v19.bin`) being in the image
means cold-start traffic and any registry-outage degradation always get *something* back — the
service is never fully dark. But the readiness probe still fails if there's no valid mounted
ranker, so a pod won't quietly serve personalized traffic without its assigned model.

Mechanically: an init container resolves the experiment arm to a model version, pulls the
artifact from `s3://recoserve-model-registry/...`, checks its SHA-256 against the registry record,
and lands it in the shared `/models` volume. The service memory-maps it on start.

This lines up with `serving/capacity-plan.md` (which sizes around a mounted ranker plus a warm
fallback) and with the architecture doc.

## The build itself

Two stages. The builder stage installs the Python deps — with `build-essential` for anything that
needs compiling — into an isolated `/install` prefix. None of those compilers make it into the
final image. The runtime stage starts from `python:3.12-slim`, copies just `/install` across,
adds the app code and the fallback model, runs as a non-root UID 10001 user, and listens on 8080.

## Size

Rough breakdown of where the ~365 MB goes:

| Layer | ~Size |
|---|---|
| `python:3.12-slim` base | 120 MB |
| ONNX Runtime (CPU) + numpy/scipy | 180 MB |
| FastAPI/uvicorn + app deps | 40 MB |
| App code + fallback model | 25 MB |
| Final image | **~365 MB** |

The ranker (~280 MB) is *not* in that total — it lives on the `/models` volume at runtime.

Things that keep it from bloating: the `-slim` base, `--no-install-recommends`, build tooling
confined to the builder stage, no model weights baked in, and a `.dockerignore` that drops tests,
docs, and notebooks.

## Security notes

Non-root user with a `nologin` shell. No build toolchain in the final image. CI pins the base by
digest and runs a Trivy scan, and the deploy is gated on zero CRITICAL/HIGH findings (see the
workflow in `cicd/`). Artifact integrity is enforced at mount time via the SHA-256 digest from
`lifecycle/model-registry.yaml`.

## Tags

The image is tagged `recsys-ranker:<semver>-<git-sha>`, e.g. `recsys-ranker:1.8.0-a1b2c3d`. That's
the code version, and it's separate from the model version (`ranker-vMAJOR.MINOR.PATCH`). Both the
CI workflow and the registry treat them as two independent things — don't try to merge them.
