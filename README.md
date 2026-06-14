![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# RecoServe — personalized in-app recommendations (Scenario X)

## What this is

RecoServe is the recommendation system behind the home-screen rail of a B2C mobile retail app.
Every time someone opens the app, it returns a ranked list of products picked for them, using
their last 30 days of browsing and purchases. It has to do that at 800 RPS peak inside a 120 ms
p95 budget measured at the phone, hand cold-start users (no history) a sensible default instead
of nothing, and let the product team run model A/B tests all day without us shipping a deploy for
each one. Models are versioned in a registry and loaded at runtime rather than baked into the
image, so swapping an experiment arm takes seconds, and every response is stamped with the model
version that produced it.

This repo is the design — architecture, lifecycle, container, API, capacity/SLOs, CI/CD,
monitoring, rollback — all built around that one scenario so the pieces actually agree with each
other.

## The diagram

Full version (request path + offline path) is in
[`architecture/architecture.md`](architecture/architecture.md). In a sentence: phone → CDN/gateway
→ recommendation service, which fans out to the online feature store (Redis) and an ANN service
for candidates, then ranks them in-process. A separate offline pipeline produces the features,
embeddings, and model artifacts on a schedule.

## The numbers that matter

| Thing | Value |
|---|---|
| Peak RPS | 800 |
| Average RPS | 280 |
| p95 budget (phone to phone) | 120 ms |
| Server-side slice of that | ~70 ms |
| Model | two-tower retrieval (ANN) + compact DLRM/GBDT ranker, ONNX |
| Ranker size | ~280 MB |
| Hardware | CPU, 2 vCPU / 4 GB pods. No GPU. |
| Replicas | 12 baseline, autoscale to 20 |
| Model delivery | mounted from the registry at startup, not baked in |
| Availability target | 99.9% over 30 days |
| Rough monthly cost | ~$5,300 |

Every other doc in here uses these same numbers. The latency breakdown and the replica math are
in [`serving/capacity-plan.md`](serving/capacity-plan.md); the formal targets are in
[`serving/slos.yaml`](serving/slos.yaml).

## Where everything lives

- Architecture — [`architecture/architecture.md`](architecture/architecture.md),
  [`JUSTIFICATION.md`](architecture/JUSTIFICATION.md), and the two [ADRs](architecture/adr/)
- Lifecycle & registry — [`lifecycle/lifecycle.md`](lifecycle/lifecycle.md),
  [`model-registry.yaml`](lifecycle/model-registry.yaml)
- Container — [`container/Dockerfile`](container/Dockerfile),
  [`container/README.md`](container/README.md)
- API — [`api/openapi.yaml`](api/openapi.yaml) plus payloads in [`api/examples/`](api/examples/)
- Capacity & SLOs — [`serving/capacity-plan.md`](serving/capacity-plan.md),
  [`serving/slos.yaml`](serving/slos.yaml),
  [`serving/load-test-plan.md`](serving/load-test-plan.md)
- CI/CD — [`cicd/.github/workflows/deploy-model.yml`](cicd/.github/workflows/deploy-model.yml)
- Monitoring — [`monitoring/alerts.yaml`](monitoring/alerts.yaml)
- Rollback — [`runbooks/rollback.md`](runbooks/rollback.md)

## Things I'd want to nail down with the team on Monday

These are the assumptions I made that I'm not fully sure about, and they'd change real decisions:

1. **How much of our traffic is actually cold-start?** I assumed ~12%. If it's much higher, the
   fallback path needs more attention and the cache-hit assumptions in the capacity plan shift.
2. **Who owns experiment assignment?** Does the product team's A/B platform hand us the arm in a
   header, or do we hash the user id ourselves? Right now the API takes an optional arm hint and
   falls back to server-side hashing, but I'd rather not guess.
3. **How fresh do the 30-day signals need to be?** I assumed a 15-minute micro-batch is fine. If
   they need sub-minute, the online store needs true streaming, and the drift baseline changes.
