---
generated: '2026-09-19'
method: generated
name: Discover models and pin a version
description: >-
  List the System One models and aliases an account may call, understand what an alias will do to
  tuned thresholds, and pin a versioned model ID so answers stop moving underneath you.
api: openapi/typesafe-ai-openapi.yml
operations: [models_v1_v1_models_get]
source: >-
  Grounded in openapi/typesafe-ai-openapi.yml (OpenAPI 3.1.0, captured live from
  https://api.typesafe.ai/openapi.json); operationId verified verbatim in the spec. Alias semantics
  and pinning guidance from https://docs.typesafe.ai/models, recorded in
  lifecycle/typesafe-ai-lifecycle.yml.
---

# Discover models and pin a version

## Auth
`Authorization: Bearer <API_KEY>`, base `https://api.typesafe.ai`. Same key as the evaluation
endpoint — there are no scopes, so there is no read-only credential for this.

## Steps

1. **List what you may call** — `models_v1_v1_models_get` (`GET /v1/models`). Returns
   `{models: [{name, description, release_date}]}`. It currently lists the ALIASES.

2. **Know that the list is not the whole set.** A versioned ID such as `jev-1.13.0` is accepted in
   the `model` field whether or not it appears in this list. So `GET /v1/models` tells you what is
   advertised, not what is callable — do not treat it as a validation oracle.

3. **Understand the two aliases before you use one.**
   - `jev-latest` → the most recent stable, official release. The SDK default and the docs default.
   - `jev-preview` → the most recent release, official or not. Moves ahead of `jev-latest` when a
     preview build exists. At the time this skill was written both resolve to the same build.

4. **Decide: alias or pin.** An alias moves when a release ships, and the answers behind it can then
   change with no change on your side. If any business logic thresholds on `confidence` or on a
   `score` value, **pin the versioned ID** and move deliberately. If you just want the best current
   model, use `jev-latest`.

5. **Always log `response.model`.** Every evaluation response reports the versioned ID that actually
   answered. That is your only audit trail for which build produced a decision, and it is the thing
   you will want when a threshold starts behaving differently.

6. **Re-read the limitations page when you move versions.** TypeSafe publishes a per-version
   known-weak-spots page (for example `/model-jaggedness/jev-1.13`). Read the new version's page
   before repointing production at it.

## Errors
- `401` — missing or invalid key.
- `422` — declared on this operation in the spec, but it takes no parameters; this is a FastAPI
  artefact rather than a reachable condition.
- The SDKs also surface `NotFoundError`, `PermissionDeniedError` and `InternalServerError`, none of
  which appear in the published server-side error table. Handle them defensively.

## Notes
- There is no deprecation policy and no Sunset/Deprecation header, so a retirement will not be
  signalled in-band. The status page Atom feed (https://status.typesafe.ai/feed.xml) carries
  incidents only, not releases; the SDK changelogs are the closest thing to a release feed.
