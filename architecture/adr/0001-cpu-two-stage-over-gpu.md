# ADR-0001: CPU two-stage recommender, not GPU single-stage scoring

Status: Accepted (2026-06-14)
Deciders: ML platform lead, serving SRE, product/experimentation

## The problem

The home rail has to render on every app open. That's 800 RPS at peak and a 120 ms p95 budget
measured at the user's phone. Once you subtract the mobile network round trip (call it 35 ms)
and the gateway/serialization overhead, we're left with about 70 ms on the server to actually
produce a recommendation.

So the real question is: how do we score products inside 70 ms? Two broad options.

1. One big model that scores the whole catalog per request. Best relevance ceiling, but you're
   not doing that over a real catalog in 70 ms without GPUs.
2. Split it. A cheap retrieval step pulls a few hundred candidates, then a smaller ranker scores
   only those. The expensive embedding/index work happens offline.

This is the decision that locks in our hardware, our cost, and how the thing autoscales, so it's
the one I want on record first.

## What we're doing

Two-stage. Two-tower ANN retrieval grabs the top 500 candidates, then an ONNX ranker (compact
DLRM / GBDT) scores just those in-process, on CPU.

## Why I'm comfortable with it

Per-request cost is basically flat. We always retrieve 500 and rank 500, so latency doesn't blow
up for power users with huge histories. Flat cost is what makes the replica math in
`serving/capacity-plan.md` defensible (12 baseline, 20 max) and keeps us in the ~$5,300/month
envelope. CPU pods also scale up in seconds, which matters more than it sounds — GPU autoscaling
is slow and the cold-start model-load penalty is real.

## What it costs us

The honest downside: we can only recommend what the ANN actually retrieved. If the index doesn't
surface a relevant item, the ranker never sees it. That's a recall ceiling. We watch recall@100
offline and it's a hard promotion gate in `lifecycle/model-registry.yaml`, so at least we'd
notice it regressing.

The ranker is also boxed in by that ~35 ms slice of the budget, so we're not picking the model
with the best offline AUC — we're picking the best one that fits the latency slice. That's a
deliberate trade, not an oversight.

If relevance turns out to be not good enough and we genuinely need a heavier ranker, the escape
hatch is a GPU ranking tier behind the same retrieve→rank interface. But that's a new ADR and a
fresh cost conversation, not something we slip in quietly.
