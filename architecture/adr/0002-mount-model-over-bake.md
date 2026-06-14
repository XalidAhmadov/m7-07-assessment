# ADR-0002: Mount model artifacts from the registry instead of baking them in

Status: Accepted (2026-06-14)
Deciders: ML platform lead, serving SRE, product/experimentation

## The problem

Product wants to A/B test models continuously. Not "once a sprint" — continuously, with arms
turning over inside a day. Each arm is a model version.

If the model lives inside the container image, every new arm means: build image, scan it, roll
it out. That ties model cadence to deploy cadence, and it makes same-day experiment turnover
basically impossible. The whole point of the experimentation program dies on the deploy queue.

## What we're doing

Keep the image model-agnostic. The only model baked in is the cold-start fallback. The real
ranker gets pulled at runtime.

Concretely: an init container reads which model version(s) the experiment config wants, downloads
the artifact from the registry's object store, checks the SHA-256 digest against what's recorded
in `lifecycle/model-registry.yaml`, and drops it in a shared `/models` volume. The service
memory-maps it on start, and stamps the version it loaded onto every response as
`X-Model-Version`.

## Why this is the right call here

Spinning up a new arm becomes a config change. No build, no scan, no rolling deploy. One image
serves every arm, which also means a smaller image and a much smaller thing to keep patched.

Rolling a model back is then independent from rolling code back — you flip the version in config
and the pods remount. The rollback runbook leans on exactly this; it's why model rollback is a
sub-five-minute operation.

## The parts that worry me, and what we did about them

Pod startup now depends on the registry being up and the artifact being intact. That's a new
failure mode on the path to readiness, and I don't love it. Three mitigations:

- The init container verifies the digest before the pod is allowed to come up. A corrupt or
  truncated download fails closed.
- The readiness probe fails if no valid ranker is loaded, so a pod never quietly serves traffic
  without its assigned model.
- The fallback model is baked in. So even if the registry is unreachable or an arm is misrouted,
  cold-start traffic and degraded serving still return something. We're never fully dark.

The other thing: provenance is now one hop away from the image. You can't just look at the image
tag and know the model. But `X-Model-Version` on every response plus the lineage in the registry
means we can trace any served prediction back to the exact artifact, so I think we come out ahead
on auditability, not behind.

One naming note so nobody trips on it later: image tags are `recsys-ranker:<semver>-<git-sha>`
(that's the *code*), and model versions are `ranker-vMAJOR.MINOR.PATCH` (that's the *artifact*).
They're separate on purpose. The CI workflow and the registry spec both treat them as two
independent things.
