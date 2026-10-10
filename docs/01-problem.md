# 01 — The problem: scans that took 29 seconds

## Symptom

A user photographs a receipt. The app sends it to the backend, the backend calls
the Gemini API, and the result comes back as structured JSON.

It took **22–29 seconds**. Sometimes longer.

For a flow whose entire promise is "point your camera and forget about it", that
is not a slow feature. It is a broken feature: people assumed the app had hung.

## Why the cause was non-obvious

Three things made this harder to attack than it looks:

1. **The delay was invisible in the logs.** The server logged a start and a
   finish. Nothing in between, and nothing that looked like an error.
2. **It was intermittent in a misleading way.** Local development was fast.
   Production was slow. That pattern usually means "production data is bigger"
   or "the free tier is throttling".
3. **Every plausible culprit had a story.** A large multimodal model *is* slow.
   A free API tier *is* throttled. A cold process *does* pay a startup cost.

The temptation is to pick the most believable story and start optimising.
I started by measuring instead.

## What it was NOT

Ruled out in this order — each one by measurement, not by reasoning:

| Hypothesis | How it was tested | Result |
|---|---|---|
| The model is slow | Same image and prompt, called directly | ✗ not the cause |
| The prompt is too large | Token count logged per call | ✗ not the cause |
| The image is too big | Resized from ~3 MB to ~300 KB | ✗ not the cause |
| My code is slow | Each stage timed server-side | ✗ not the cause |
| The API quota is throttling | Usage counters checked | ✗ not the cause |
| **The host's network path** | **Same request from two origins** | ✅ **this one** |

## The measurement that broke it open

The decisive test was not clever. It was the same request issued from two
places, then compared:

| Origin | Time to reach the AI provider |
|---|---|
| The production host | **~29 s** |
| My laptop, same request | **~3.5 s** |

Same key, same payload, same model. An **8× difference** that depended only on
*where the request came from*.

That single comparison eliminated the application, the model and the prompt, and
left exactly one variable: the network path between host and provider.

## Why this matters as a habit

The expensive mistake here is not a slow scan. It is **optimising the wrong
layer**: rewriting prompts, shrinking images, adding caches — real work, none of
it addressing the cause.

A five-minute A/B measurement replaced weeks of speculative optimisation. It
also produced a fix that lives in the *code* rather than in host configuration,
so it cannot silently regress when the deployment target changes.

→ Next: [02 — The diagnosis](02-diagnosis.md)
