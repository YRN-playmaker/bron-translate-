# Multi-Engine ASR Routing

> Pseudocode + design notes. Engine names, model files, endpoints, and tuning
> constants are abstracted — this shows the *routing architecture*, not the
> production wiring.

## Problem

A real-time speech-to-text gateway running on **constrained hardware** (a
single-board server) has to balance three things that pull in opposite
directions:

- **Cost** — cloud ASR APIs charge per minute; local inference is free but
  CPU-bound.
- **Accuracy** — heavier models transcribe better but are slower.
- **Latency** — real-time captioning can't wait seconds per utterance.

No single engine wins on all three. The answer is a **pluggable engine layer**
that routes each session to the right backend based on the user's tier and
selected mode.

## Engine abstraction

Every ASR backend — whether a local model or a remote API — implements one
interface, so the pipeline never special-cases a specific engine:

```
interface ASREngine:
    transcribe(audio_chunk) -> { text, is_final }
    # local engines run in-process; remote engines wrap an HTTP/streaming call
```

Concrete engines (abstracted here):

```
LocalLightweight   # fast, low-resource, runs on CPU; lower accuracy
LocalMedium        # better accuracy, heavier; gated behind a concurrency lock
RemoteCloud        # highest accuracy + multilingual; metered, costs money
```

## Routing

Selection is driven by **tier + mode**, decided once at session start:

```
function select_engine(tier, requested_mode, system_load):
    if requested_mode == LOCAL and tier.allows_local:
        return LocalLightweight        # offload compute, zero marginal cost

    if tier.is_premium:
        return RemoteCloud             # premium pays for best quality

    if requested_mode == MEDIUM and tier.allows_medium:
        if medium_lock.is_free():      # protect the box from overload
            return LocalMedium
        else:
            return LocalLightweight    # graceful fallback under contention

    return LocalLightweight            # safe default
```

### Concurrency guard

The heavier local model can starve a small server if too many sessions grab it
at once. A lock caps how many concurrent sessions may use it; everyone else
degrades gracefully to the lightweight engine instead of crashing the box:

```
function acquire_medium_or_fallback(token):
    if token in active_medium_sessions:
        return LocalMedium             # already holding the slot
    if len(active_medium_sessions) < MEDIUM_SLOT_LIMIT:
        active_medium_sessions.add(token)
        return LocalMedium
    return LocalLightweight            # over capacity -> fall back
```

## Real-time pipeline

Audio arrives over a WebSocket as a stream of PCM chunks. The pipeline feeds
the selected engine and emits partial vs. final results separately, so the UI
can show live "interim" text that firms up into final captions:

```
on websocket connect:
    engine = select_engine(tier, mode, load)
    register_concurrency(token)

loop while connected:
    chunk = await next_audio_chunk()
    result = engine.transcribe(chunk)

    if result.is_final:
        emit("final", result.text)
        spawn translate_and_meter(result.text, token)   # async, non-blocking
    else:
        emit("partial", result.text)                     # live preview

on disconnect (always):
    release_concurrency(token)        # critical: every exit path must release
    cleanup_buffers()
```

> The `on disconnect` release runs in a `finally`-equivalent block so that a
> dropped or errored connection never leaks a concurrency slot — a subtle bug
> that otherwise locks users out over time.

## Why this design

| Decision | Rationale |
|---|---|
| One `ASREngine` interface | Pipeline is engine-agnostic; adding a backend = one new class |
| Tier-driven routing | Cost control: free/local for casual use, paid cloud for premium |
| Local-by-default, cloud-on-demand | Keeps marginal cost near zero on a single-board server |
| Concurrency lock on heavy local model | Prevents OOM / overload from too many simultaneous heavy sessions |
| Graceful fallback (never hard-fail) | Under contention, users get *lighter* transcription, not an error |
| Partial vs. final separation | Enables live interim captions without waiting for end-of-utterance |
| Release slot in `finally` | Guarantees no slot leak on abnormal disconnects |

## Extensibility note

This is the seam where a **client-side GPU engine** would slot in: a future
backend that runs a multilingual model on the *user's* hardware, pushing the
heaviest compute entirely off the server while reusing the same interface and
routing logic unchanged.
