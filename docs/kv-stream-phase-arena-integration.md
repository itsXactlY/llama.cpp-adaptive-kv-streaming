# Adopting `kv-stream-phase-arena`: what it actually takes

Status: **investigated, not started.** Written 2026-09-12. `feature/kv-stream-phase-arena`
now exists as a real local + fork branch (`ae09597ff`) alongside `feature/adaptive-kv-stream`
(main) — brought in for reference and future work, not merged.

## What it is

67 commits by Raymond Huang (the upstream owner, `origin/RaymondHuang210129` — not
either Claude session, and not part of tonight's work), dated 2026-08-23 through
2026-09-01, all predating this fork's own recent history. It replaces this fork's
block KV streaming resource model outright:

| | This fork today (`feature/adaptive-kv-stream`) | phase-arena |
|---|---|---|
| Flag | `--kv-stream-stage-mib N` | `--kv-stream-arena-mib N` (stage-mib kept as a compat alias) |
| What N buys | A fixed **resident KV pool**, separate from compute buffers | One shared **physical arena** for phase-specific compute workspace + resident KV pages + the transfer ring |
| During prefill | Compute buffers allocated independently of the KV pool budget | Scheduler borrows workspace it needs from the arena; KV keeps a small nonzero ring |
| During decode | Same fixed KV pool as prefill | Large prefill workspace is released back; reclaimed bytes become additional resident/ring KV capacity |
| Device-memory auto-fit | Standard llama.cpp dry-run estimator | Bypassed when arena is nonzero — the estimator can't represent overlapping phase lifetimes, so real init measures both phase graphs directly and rejects an arena too small for either |

The pitch, in the phase-arena README's own words: the arena value is "the total
shared allocation, not just KV capacity" — prefill and decode stop being sized as
if they both need their peak VRAM simultaneously, because they don't run at the
same time. That is a real, structural improvement over today's model, where the
resident KV pool and compute buffers are sized independently and both provisioned
for worst case at once.

**Why this matters for tonight specifically:** the entire VRAM-tightness saga
(vision vs. DFlash2 vs. `MAIN_KV_POOL`, the 423MB-free scare) exists because compute
buffers and the resident KV pool compete for the same fixed budget with no way to
time-share it. Phase-arena's whole design is exactly the fix for that class of
problem. This is a real reason to want it, not just a nice-to-have refactor.

## Why it's not a merge — checked, not assumed

`git merge-tree` (read-only simulation, nothing touched) against current main:
**28 conflicting files**, several as add/add — both branches independently
invented `src/llama-kv-stream-config.{h,cpp}` after diverging from the same
ancestor (`ece963f41`, confirmed absent there), converging on nearly the same
shape (`llama_kv_stream_config`, `llama_kv_stream_pool_layout_params`) but
different field names (`arena_bytes`/`minimum_arena_bytes` vs. this fork's
`stage_bytes`/`minimum_stage_bytes`). That convergence is a good sign for
eventual integration — the concepts didn't diverge, only the implementation —
but it means a mechanical `git merge` would produce garbage, not a usable result.

Files in the conflict set that tonight's session specifically depended on:

- `src/llama-kv-cache.cpp`, `src/llama-memory-hybrid.cpp` — carry the IMROPE
  `get_can_shift()` fix (commit `bad7199cd`). Phase-arena modifies both.
- `tools/server/server-context.cpp` — restructured twice tonight (`pre_decode()`
  then `decode()`/`post_decode()` into pipeline classes) and already carries the
  `abort_all_slots()` OOM-crash fix. Phase-arena modifies this file extensively too.
- `src/llama-kv-stream-config.{h,cpp}` — add/add, as above.
- `common/arg.cpp`, `common/common.{h,cpp}` — CLI flag wiring, straightforward
  content conflicts but numerous.
- The CUDA layer (`ggml-cuda.cu`, `fattn*.cu(h)`) — the actual arena/buffer
  implementation. This is where the real engineering is; conflicts here aren't
  cosmetic, they're two different memory-management strategies for the same
  tensors.

## Why a `git merge` is the wrong tool here

A conflict-resolution pass over 28 files where two independent implementations
of the same subsystem collide would produce something that compiles by
coincidence, not by design — exactly the kind of change too large to actually
review, on code this fragile (CUDA memory lifetime bugs are silent until they
aren't). The right approach is a **rebase-and-adapt**, not a merge:

1. Start from current `feature/adaptive-kv-stream` (main), which already has
   tonight's fixes.
2. Read phase-arena's 67 commits as a *reference design*, not a patch to apply.
   The commit sequence itself is a good implementation order to mirror:
   `ba25c82bf` (partition planner) → `8f1a683f3` (shared buffers) →
   `1537434ea` (bind KV streaming to arena) → `283ccf2da` (repartition by graph
   phase) → `cf50bf191` (signal decode phases explicitly) → `ae09597ff`
   (finalize interface).
3. Re-implement each step directly against the current tree, porting logic by
   hand rather than cherry-picking — this is what makes the add/add config
   files resolvable at all: write the *current* fork's version of
   `llama_kv_stream_config` extended with arena semantics, not a merge of two
   incompatible structs.
4. At each step, re-run the IMROPE guard and the `abort_all_slots` crash-path
   test manually — phase-arena's changes to `llama-kv-cache.cpp` /
   `llama-memory-hybrid.cpp` need to be checked against `get_can_shift()`'s
   current logic specifically, since that fix landed after phase-arena's last
   commit and phase-arena has never seen it.
5. Re-verify `server-context.cpp` compiles and the two pipeline-refactor
   commits' structure (phase-based `pre_decode_pipeline` /
   `decode_pipeline`/`post_decode_pipeline`) still makes sense once arena
   phase-borrowing is threaded through it — these may turn out to be
   complementary abstractions (both are literally "phases"), worth checking
   for that synergy deliberately rather than as an accident.

## Recommended sequencing

Not a single session. Suggested checkpoints, each independently mergeable to
main and independently testable:

1. Port `llama_kv_stream_config`/`llama_kv_stream_pool_layout` as an
   *extension* of the current struct (arena fields alongside stage fields,
   `--kv-stream-stage-mib` kept working as an alias, matching phase-arena's
   own compat approach) — no behavior change yet, just the data model.
2. Port the CUDA-side shared-arena buffer management
   (`ggml_backend_cuda_kv_stream_*`) as a new code path selected only when
   `--kv-stream-arena-mib` is explicitly passed, `--kv-stream-stage-mib`
   continuing to use the current fixed-pool path unchanged. Both paths
   coexist; nothing currently working regresses.
3. Port the phase-signaling (`server-context.cpp`'s decode-phase hooks) once
   (1) and (2) are proven, using the already-refactored pipeline classes as
   the integration point.
4. Only once all three check out under real load (specbench, `monitor.py`)
   does `--kv-stream-arena-mib` become the recommended default in `stack.conf`
   over `--kv-stream-stage-mib`.

## Open question worth resolving before starting

Is `feature/kv-stream-phase-arena` actually finished upstream, or WIP? Its
last commit (`ae09597ff`, "finalize shared phase arena interface") reads as
complete, but it predates this fork's most recent divergence entirely and
was never rebased forward — worth a quick check of Raymond's own repo/PRs for
whether it shipped, stalled, or was itself later revised, before treating it
as a stable reference design to port from.
