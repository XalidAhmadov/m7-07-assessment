# Rollback runbook

Every
trigger below is one of the alerts in `monitoring/alerts.yaml`, so whatever paged you maps to a
section here.

The thing to remember: models are mounted, not baked (ADR-0002). So most of the time the fastest
fix is flipping a model version in config, *not* redeploying code.

## Quick triage

- Paged by a model-version or quality/drift alert → it's the model. Go to **A**. Seconds.
- Paged by availability burn or latency, and there was a recent **code** deploy → go to **B**.
- Paged by availability burn with nothing recently changed → a dependency is sick. Go to **C**.

## The triggers (these match monitoring/alerts.yaml exactly)

- **1 — availability fast burn:** `AvailabilityFastBurn`, 14.4× over 1h/5m. Page.
- **2 — availability slow burn:** `AvailabilitySlowBurn`, 6× over 6h/30m. Page.
- **3 — latency:** end-to-end p95 over 120 ms for 10 minutes. Page.
- **4 — version mismatch:** `ModelVersionMismatch` / `UnexpectedModelVersionServing` — what we're
  serving isn't the version the arm should be on. Page.
- **5 — drift / quality:** `FeatureDriftHigh`, `ColdStartRateSpike` (over 25%), or
  `RecommendationQualityDrop` (CTR more than 5% below control). Page or ticket.

---

## A — Roll back the model (no deploy, fastest path)

This is for trigger 4 or 5, where the model itself is the problem.

1. Check what's actually live: `curl -sI $PROD/reco/v1/readyz | grep -i x-model-version`, or look
   at the `served_requests_total` by `model_version` panel.
2. Find the target to roll back to — the prior version with `rollback_eligible: true` in
   `lifecycle/model-registry.yaml`. Right now that's **`ranker-v2.3.0`**.
3. Point both arms at the safe version:
   ```
   ./deploy.sh --env production --set-model-version ranker-v2.3.0 --all-arms
   ```
   That's a config change. The pods re-resolve and remount — no image build.
4. Confirm: responses now show `X-Model-Version: ranker-v2.3.0`, and the mismatch / quality alert
   clears within ~5 minutes.
5. If a quality alert is what triggered this, freeze promotions (per the error-budget policy in
   `serving/slos.yaml`) until someone's found the root cause.

Target time: under 5 minutes.

## B — Roll back the code

For trigger 1, 2, or 3 right after a service deploy.

1. Grab the last-good image tag (`recsys-ranker:<semver>-<git-sha>`) from the deploy history.
2. Point prod back at it:
   ```
   ./deploy.sh --env production --image ghcr.io/example/recoserve/recsys-ranker:<last-good> \
     --strategy rollback
   ```
3. Leave the model version alone — code rollback and model rollback are independent (that's the
   whole point of A being separate).
4. Confirm p95 back under 120 ms and 5xx back under 0.1% within ~10 minutes.

## C — A dependency is degraded

Availability or latency burning with no recent change usually means something downstream.

- **Redis (features):** check node health and `feature_store_ingest_lag_seconds`. If a node's
  down, the service already auto-degrades to the cold-start fallback and should stay inside the
  availability SLO — fail over / replace the node.
- **ANN service:** if retrieval errors spike, scale ANN replicas. Ranking degrades to fallback if
  retrieval is fully unavailable.
- **Saturation:** if CPU's pinned, check HPA actually scaled toward 20 (`serving/capacity-plan.md`).
  If it's lagging a spike, bump min replicas manually.

---

## Once it's calm again

- Drop a note in the incident channel: what triggered, what you did, the current
  `X-Model-Version`, the time.
- Make sure the alert that paged you has actually cleared in `monitoring/alerts.yaml`.
- File the follow-up: root cause, and whether the load test (`serving/load-test-plan.md`) should
  have caught this before users did.
- If you rolled a model back, demote the bad version to `Archived` in the registry with a note, so
  the lineage stays honest.
