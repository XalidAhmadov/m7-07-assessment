# Why this architecture

The short version: recommendations have to show up the instant the app opens, so the request
path has to be fast and synchronous, and everything expensive has to be done ahead of time. Most
of the design falls out of that one fact plus the experimentation requirement.

## What the scenario actually forces

A few constraints aren't negotiable, and each one rules things in or out:

- **800 RPS, 120 ms p95.** No heavy compute on the request path. Features and candidates have to
  be precomputed, and inference has to be cheap enough to run on CPU.
- **Personalization from 30 days of history.** You can't recompute a 30-day window per request,
  so an online feature store is mandatory, not optional. We look up, we don't aggregate.
- **Cold start has to still work.** Some users have no history. They need a separate, always-warm
  fallback (popularity/segment) that kicks in when the feature lookup comes back empty.
- **Continuous A/B.** Models are versioned in a registry and mounted at runtime, picked by
  experiment arm. Every response says which version produced it. See ADR-0002 for the gory bits.

## The pattern

Online synchronous serving with a two-stage recommender — retrieve, then rank — sitting on top
of precomputed features and embeddings, running on CPU.

The reason this works is that the genuinely expensive parts of recommendation (building
user/item embeddings, building the candidate index, training) have no per-request dependency you
can't precompute. So we shove all of it offline. At request time the work is bounded: one feature
lookup, one ANN retrieval that returns a fixed 500, one ranking pass over those 500. Cost per
request is flat and predictable, which is exactly what a tight p95 needs.

## Things we looked at and didn't pick

**Precompute every user's recs offline and just serve from a cache.** Cheapest, fastest. But it
can't react to anything in-session, it makes A/B testing clumsy, and a cache covering tens of
millions of users is big and goes stale fast. Killed by the experimentation requirement — though
we do reuse the idea for the cold-start fallback, where stale is fine.

**One deep model scoring the whole catalog per request.** Highest relevance ceiling in theory.
Blows the latency budget and drags in GPUs. No. (More in ADR-0001.)

**GPU serving.** Overkill for a compact ranker over 500 candidates. Adds cost, slow autoscaling,
and model-load latency for zero p95 benefit at this candidate count. Rejected — ADR-0001.

**Bake the model into the image.** Tempting because provenance is dead simple. But it turns every
experiment arm into a deploy, which is a non-starter for continuous A/B. Rejected — ADR-0002.

## The trade-offs I'd own in a review

1. Mounting models adds a registry dependency to pod startup. I accept it because experimentation
   is a first-class requirement, and we fence it with digest verification, a readiness gate, and a
   baked-in fallback so we're never fully dark.
2. Two-stage retrieval can miss good items the index didn't surface. We trade a bit of recall
   ceiling for predictable latency, and we gate on recall@k offline so it can't silently rot.
3. CPU caps how fancy the ranker can be. Fine. The ranker is sized to the budget and the
   500-candidate set, not to a leaderboard.

## Where this connects to the rest of the repo

If any of these drift apart, the submission stops being coherent, so they're worth stating plainly:

- 800 RPS and 120 ms here are the same numbers in `serving/slos.yaml` and
  `serving/capacity-plan.md`.
- "Mounted, not baked" here matches the Dockerfile (no model copied in) and `container/README.md`.
- The `X-Model-Version` response stamp shows up in `api/openapi.yaml`, gets checked in
  `monitoring/alerts.yaml`, and is what the rollback runbook flips.
- The image/model tag schemes here are the ones the CI workflow and the registry both use.
