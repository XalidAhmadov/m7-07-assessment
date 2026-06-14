# Capacity plan

Same numbers as everywhere else: 800 RPS peak, 280 average, 120 ms p95 measured at the phone.

## Where the 120 ms goes

The budget is end-to-end, so the network counts against us even though we don't control it. Here's
how I'm spending it:

| Segment | ms | Note |
|---|---|---|
| Phone ↔ edge round trip | 35 | Not ours to optimize, but it's real, so it's in the budget |
| Gateway (TLS, auth, routing) | 5 | Managed |
| Feature fetch from Redis | 15 | Store targets p99 < 10 ms; I budgeted 15 to leave room for a retry |
| ANN retrieval (top 500) | 20 | |
| Ranking 500 candidates on CPU | 35 | ONNX Runtime, in-process |
| Serialization + experiment stamping | 10 | |
| **Total** | **120** | matches the SLO |

Knock the 35 ms network off and we've got ~85 ms on the server. I'm holding the team to a 70 ms
p95 internal target so there's headroom — and not coincidentally, 70 ms is the same number we gate
canaries on in `lifecycle/model-registry.yaml` (the shadow `server_p95_ms` check).

## How much one replica can take

Each pod is 2 vCPU / 4 GB, ONNX Runtime with the thread pool pinned to 2, uvicorn running 2
workers. Measured service time per request lands around 50 ms (feature + ANN + rank).

If I want p95 under 70 ms I can't run the pod hot, so I cap utilization around 60%. Theoretical
ceiling is 2 workers / 0.050 s ≈ 40 req/s; at 60% that's roughly **65 RPS per replica** that I'm
willing to actually plan around.

## Replica count

```
800 / 65 = 12.3  → 13 to cover peak
+1 for rolling updates and zone spread → 14
HPA targets 60% CPU
```

So: **12 replicas baseline**, spread 4-per-zone across 3 zones, which means losing a whole zone
still leaves enough to cover average load. **HPA min 12, max 20.** Twenty replicas is 20 × 65 =
1,300 RPS, about 1.6× peak, which is the cushion for spikes and partial degradation. (12/20 is the
same pair in the README.)

Scaling triggers on CPU over 60% or queue depth backing up. Scale-up settles in 30 s; scale-down
waits 300 s so it doesn't flap every time traffic breathes.

## Dependencies

| Dependency | Sizing | Headroom |
|---|---|---|
| Redis (online features) | 3 nodes, ~1,000 ops/s at peak (800 RPS × ~1.2 lookups) | ~3× |
| ANN service | 2 replicas holding the ~1.2 GB index, 800 QPS | ~2× |
| Model registry / object store | read only at pod startup, never on the hot path | n/a |

## A note on cold start

I assumed ~12% of requests are cold-start (that's open question #1 in the README). Those hit the
baked-in fallback in under 5 ms and skip ANN and ranking entirely, so they're *cheaper* than warm
requests. Since I sized the replica math on the warm path, that assumption is conservative — if
cold-start is higher than 12%, we have more headroom, not less.

## Cost, roughly

| Item | ~Monthly |
|---|---|
| Serving pods (~14 avg, 2 vCPU/4 GB) | $2,600 |
| Redis (3 nodes) | $900 |
| ANN service (2 replicas) | $700 |
| Object store + registry + egress | $300 |
| Observability (Prom/OTel/Loki + retention) | $800 |
| **Total** | **~$5,300** |

Lines up with the README. No GPU line item, on purpose (ADR-0001).
