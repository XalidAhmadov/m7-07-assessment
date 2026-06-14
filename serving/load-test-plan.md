# Load test plan

## What we're trying to prove

That the service actually holds the SLOs in `serving/slos.yaml` — 120 ms p95 at 800 RPS, 99.9%
availability — and that the replica math in the capacity plan (12 baseline, scaling to 20) is real
and not just arithmetic. A load test that fires the same request for the same user at one endpoint
proves nothing here, because it would sit entirely in cache and never touch the cold path. So most
of the effort below is in making the *traffic* look real.

## Making the traffic realistic

| Knob | Setting | Why it matters |
|---|---|---|
| Endpoint mix | 95% sync `/recommendations`, 4% `:batch`, 1% async submit | Hot path dominates; batch/async can't be allowed to starve it |
| User ids | ≥ 2M distinct, Zipfian reuse | Gives a realistic feature-store hit/miss mix instead of a fake 100% hit rate |
| Cold-start share | 12% unknown ids | Exercises the fallback path the capacity plan assumes |
| `num_results` | 20 | What the home rail asks for |
| Arrival model | open (rate-based), not closed | Reproduces RPS rather than a fixed concurrency, which is what we actually care about |

## The stages

1. **Smoke** — 10 RPS for a couple of minutes. Just confirming correctness, that the headers come
   back (`X-Model-Version`, `X-Request-Id`), and that cold start works before we lean on it.
2. **Steady peak** — ramp to 800 RPS, hold 30 minutes. Pass = p95 ≤ 120 ms, errors < 0.1%, HPA
   settling around 12–14 replicas.
3. **Find the knee** — push 800 → 1,300 RPS. Confirms the 20-replica ceiling absorbs ~1.6× peak,
   and tells us the RPS where p95 finally crosses 120 ms.
4. **Spike** — jump 280 → 1,000 RPS instantly. Watching whether scale-up keeps up (30 s
   stabilization) and, more importantly, whether anything 5xxes during the scramble. We want
   graceful queueing, not dropped requests.
5. **Soak** — 800 RPS for 2 hours. This is the one that catches leaks, Redis connection
   exhaustion, and GC pauses showing up in the tail. p99 ≤ 250 ms has to hold the whole time.
6. **Break a dependency** — kill a Redis node or an ANN replica mid-run. Confirms we degrade to
   the fallback within the availability SLO and recover cleanly afterward.

## Setup

k6 with the `constant-arrival-rate` executor (Locust would also do), pointed at the gateway URL.
Run it against a staging cluster shaped like production — same 2 vCPU/4 GB pods, same 12/20 HPA,
same Redis and ANN topology — with a synthetic catalog and a seeded feature store. The model is
*mounted*, not baked (ADR-0002), so we also get a real read on mount/startup time.

## What we measure and what counts as a pass

| Metric | From | Pass |
|---|---|---|
| p95 / p99 latency | k6 + gateway | p95 ≤ 120 ms, p99 ≤ 250 ms at 800 RPS |
| 5xx rate | gateway | < 0.1% |
| Replica count | HPA / Prometheus | ~12–14 at peak, ≤ 20 under stress |
| Per-segment latency | OTel spans | each stays inside its capacity-plan slice |
| Cold-start coverage | app metric | ≥ 99.9% non-empty |
| CPU / queue depth | Prometheus | CPU ≤ 60% at steady peak |

## Done when

Stages 2 through 6 all pass. The results then feed the burn-rate thresholds in
`monitoring/alerts.yaml`, and they're how we confirm the rollback triggers in
`runbooks/rollback.md` are actually reachable before a real incident is the thing that tests them.
