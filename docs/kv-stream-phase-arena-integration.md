# Integrating `feature/kv-stream-phase-arena`

Status: investigated, not started.

## Repository provenance

This repository is a fork of [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp).
The adaptive KV streaming feature was written by Raymond Huang
(`raymond860909@gmail.com`) in
[RaymondHuang210129/llama.cpp-adaptive-kv-streaming](https://github.com/RaymondHuang210129/llama.cpp-adaptive-kv-streaming),
beginning 2026-08-20 with `ba8632fdd` ("kv-stream: add block-granular residency
planner"). Design and benchmarks are described in
[Running Qwen 27B on 16G VRAM with Full Context Length](https://medium.com/@raymond860909/running-qwen-27b-on-16g-vram-with-full-context-length-building-adaptive-kv-cache-streaming-for-bf1e819116e9).

`itsXactlY/llama.cpp-adaptive-kv-streaming` is a fork of Raymond's repository.
Both branches discussed here carry his streaming work; they differ in which
upstream base they sit on and which generation of the streaming design they use.

## The two branches

Common ancestor: `ece963f41`.

`feature/adaptive-kv-stream` (this fork's default) is 450 commits ahead of that
ancestor: roughly 380 upstream llama.cpp commits synced forward, 61 by Raymond
Huang, 9 local.

`feature/kv-stream-phase-arena` is 67 commits ahead of the same ancestor. The
first 61 are the same Raymond Huang commits that `feature/adaptive-kv-stream`
also contains. The final 6, dated 2026-08-31 to 2026-09-01, are the phase arena
work:

    ba25c82bf  kv-stream: add phase arena partition planner
    8f1a683f3  cuda: add shared phase arena buffers
    1537434ea  cuda: bind KV streaming to phase arena
    283ccf2da  kv-stream: repartition shared arena by graph phase
    cf50bf191  server: signal KV arena decode phases explicitly
    ae09597ff  kv-stream: finalize shared phase arena interface

So the branch is not a competing implementation. It is the next revision of the
same feature, developed on an older upstream base and never rebased forward.

## What the phase arena changes

| | `feature/adaptive-kv-stream` | `feature/kv-stream-phase-arena` |
|---|---|---|
| Flag | `--kv-stream-stage-mib N` | `--kv-stream-arena-mib N`; `--kv-stream-stage-mib` retained as an alias |
| What N allocates | A resident KV pool, sized independently of compute buffers | One arena shared by phase compute workspace, resident KV pages, and the transfer ring |
| Prefill | Compute buffers allocated outside the pool budget | Scheduler borrows workspace from the arena; KV keeps a small nonzero ring |
| Decode | Same fixed pool as prefill | Prefill workspace is released; reclaimed bytes become resident/ring KV capacity |
| Device-memory auto-fit | Upstream dry-run estimator | Bypassed when an arena is configured, because the estimator cannot represent overlapping phase lifetimes. Context init measures both phase graphs and rejects an arena too small for either |

The practical difference: prefill and decode never run simultaneously, so sizing
both for peak at once wastes VRAM. The arena lets decode reclaim what prefill no
longer needs. On a card where the resident pool and compute buffers are already
competing for the last few hundred MiB, that is the relevant constraint.

## Why this is not a merge

`git merge-tree` against `feature/adaptive-kv-stream` reports 28 conflicting
files. Several are add/add: both branches introduced
`src/llama-kv-stream-config.{h,cpp}` independently after `ece963f41` (the files
do not exist in the ancestor). The structures converged —
`llama_kv_stream_config`, `llama_kv_stream_pool_layout_params` — but the fields
differ (`arena_bytes`/`minimum_arena_bytes` against `stage_bytes`/`minimum_stage_bytes`).
Git cannot reconcile that; it has no basis to prefer either.

Conflicting files that carry local changes:

- `src/llama-kv-cache.cpp`, `src/llama-memory-hybrid.cpp` — hold the
  `get_can_shift()` IMROPE fix (`bad7199cd`). The phase arena branch predates it.
- `tools/server/server-context.cpp` — holds the `abort_all_slots()` slot-reset
  fix (`6f3b0a364`) and the `pre_decode()`/`decode()`/`post_decode()` pipeline
  refactors (`21e88b271`, `c258a9277`). The phase arena branch modifies this file
  substantially as well.
- `common/arg.cpp`, `common/common.{h,cpp}` — flag wiring.
- `ggml/src/ggml-cuda/*` — the arena implementation itself.

## Suggested approach

Port the design rather than merge the branch. Work from
`feature/adaptive-kv-stream`, using the six phase arena commits as a reference
implementation order, and write each step against the current tree. This is what
makes the add/add files tractable: extend the existing
`llama_kv_stream_config` with arena semantics instead of reconciling two structs.

Checkpoints, each independently buildable and mergeable:

1. Extend `llama_kv_stream_config` / `llama_kv_stream_pool_layout` with arena
   fields alongside the existing stage fields. `--kv-stream-stage-mib` keeps its
   current meaning. No behavior change.
2. Add the CUDA shared-arena buffer management as a second code path, selected
   only when `--kv-stream-arena-mib` is passed. The existing fixed-pool path stays
   as-is, so nothing currently working changes.
3. Add decode-phase signaling in `server-context.cpp`, using the existing
   pipeline classes as the integration point. Both designs are organized around
   graph phases, so check whether the pipeline structure can carry the phase
   transitions directly rather than duplicating them.
4. Re-verify `get_can_shift()` behavior after any change to
   `llama-kv-cache.cpp` or `llama-memory-hybrid.cpp`; the phase arena branch has
   never seen that fix.

Only after all four hold under benchmark load should `--kv-stream-arena-mib`
replace `--kv-stream-stage-mib` as the configured default.

## Open question

Whether `feature/kv-stream-phase-arena` is finished or abandoned upstream. The
last commit reads as complete, but it was never rebased onto the newer upstream
base. Check Raymond's repository for whether the work continued, shipped, or was
revised before treating it as a stable reference.
