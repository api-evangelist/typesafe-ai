---
generated: '2026-09-19'
method: generated
name: Evaluate state with typed questions
description: >-
  Ask TypeSafe's System One model one or more typed questions (Noul, Choice, Score) about a piece of
  content and act on the structured answers in code — including reading confidence to decide when to
  act autonomously and when to escalate.
api: openapi/typesafe-ai-openapi.yml
operations: [systemone_v1_systemone_post]
source: >-
  Grounded in openapi/typesafe-ai-openapi.yml (OpenAPI 3.1.0, captured live from
  https://api.typesafe.ai/openapi.json). The single operationId was verified verbatim in the spec.
  Auth per authentication/typesafe-ai-authentication.yml, errors per
  errors/typesafe-ai-problem-types.yml, type graph per data-model/typesafe-ai-data-model.yml,
  cross-cutting rules per conventions/typesafe-ai-conventions.yml.
---

# Evaluate state with typed questions

One request, one operation, N decisions. This is the whole product surface.

## Auth
- `Authorization: Bearer <API_KEY>`. Base URL `https://api.typesafe.ai`.
- Both official SDKs read `TYPESAFE_API_KEY` from the environment. Create a key at
  https://console.typesafe.ai/keys.
- There are no scopes. A key can call both operations and spend tokens; do not hand one to an agent
  you would not hand a budget to.

## Steps

1. **Assemble the state.** `state` is the content to judge — a plain string, or a JSON object/array
   for chat logs, records, or application state. Text only: pre-process images, audio or binaries
   into text or structured fields first.

2. **Name your questions.** `questions` is a map and YOU choose every key. The answer comes back
   under the same key. The key is not sent to the model and is not used in inference, so use whatever
   your code wants to switch on.

3. **Choose a question type per decision.**
   - `noul` — a yes/no question. `instructions` is the question (or a declarative statement to
     evaluate). Optional `criteria` is an object `{true, false}` describing what each side means.
   - `choice` — one option from a set. `criteria` is a **map** of option → description (`null` when
     an option needs no detail).
   - `score` — a rating against ordered levels. `criteria` is an **ordered ARRAY** of at least two
     level descriptions. (It was an integer-keyed dict before 2026-09-15; both SDKs changed at 0.6.0.)

4. **Call it** — `systemone_v1_systemone_post` (`POST /v1/systemone`) with `state`, `model` and
   `questions`. Use `"jev-latest"` unless you have tuned thresholds against a build, in which case
   pin the versioned ID (e.g. `jev-1.13.0`) and move on your own schedule.

5. **Batch, do not loop.** Jev ingests `state` once and evaluates every question against it in
   parallel. Send all the questions you might want — including speculative ones — in ONE call and let
   your code decide which answers to read. TypeSafe's own measurement of a 13-question briefing is
   12.2x cheaper and 10.0x faster batched, with no change in answers. Ceilings: 64k tokens for state
   plus all questions combined, 32k for state plus the single longest question.

6. **Read the answers.** `answers` is keyed by your question ids and each entry carries a `type`
   matching its question:
   - Noul → `noul` (0 = no, 1 = yes). **No `confidence` field** — the probability IS the answer.
   - Choice → `choice`, `probabilities` (map of option → float, summing to 1), `confidence`.
   - Score → `score` (probability-weighted, and it CAN land between levels, e.g. 1.7), `legend`,
     `probabilities`, `confidence`.

7. **Threshold in code, not in the prompt.** Act automatically above your confidence threshold;
   escalate to a human below it. If all you need is the best option, take the highest probability
   rather than setting a threshold at all — the provider's own guidance.

8. **Log the model that answered.** The response's `model` field reports the versioned ID that
   handled the call, not the alias you sent. Record it: `jev-latest` moves when a release ships.

## Retries and cost
- Retry `429 Too Many Requests` and `529 Overloaded` with exponential backoff. Honour `retry-after`
  when present — it is not always sent. **529 is not an IANA-registered status**; add it to your
  retryable set explicitly if you are calling HTTP directly.
- There is no idempotency key, and none is needed: the call persists nothing, so a replay cannot
  corrupt state. It CAN cost twice — input tokens are billed, output tokens are free. Bound your
  retries.

## Errors
- `401` — missing or invalid key; check the `Authorization` header.
- `422` — validation failure. Body is FastAPI-shaped: `{"detail": [{loc, msg, type, input?, ctx?}]}`.
  Walk `detail[].loc` to find the field. The classic cause here is a Score question with fewer than
  two `criteria` levels (`ctx.min_length: 1`).
- Not RFC 9457 — there is no `type` URI to dereference. See `errors/typesafe-ai-problem-types.yml`.

## Notes
- Do not use TypeSafe for open-ended generation, creative writing or multi-step reasoning. It returns
  judgments, not text.
- Put your questions and threshold constants in ONE file. They are the part a human must review.
