# Parameters

Set via `io.SetParameters({...})`; the `BBSTAGE_*` environment fallbacks
(see [Quickstart](quickstart.md)) exist for middleware paths. Precedence:
**parameter > environment > default**.

| Parameter | Default | Meaning |
|---|---|---|
| `BurstBufferPath` | `$TMPDIR` or `/tmp` | Node-local staging directory. The node's local NVMe disk (`/tmp`) is the standard and recommended tier; see the tier footnote in [Tuning](tuning.md#staging-tier) for the only exception. |
| `CloseMode` | `durable` | `durable` or `staged`. Determines what `Close()` waits for — see [Semantics](semantics.md). |
| `StagingWatermarkMB` | `1024` | Staged mode: cap on staged-but-undrained bytes **per rank**. `EndStep` blocks above it until the concurrent drain catches up; after `CommitTimeoutSecs` without drain progress it throws (0 = unlimited). The engine's backpressure valve: bounded, visible waits instead of unbounded growth. |
| `DrainThreads` | `1` | Drain worker threads per rank (started from `Open` in both close modes). The node's drain width = threads × ranks-per-node, bounded by store concurrency scaling. |
| `CommitTimeoutSecs` | `300` | The watchdog — leave at default. Maximum wait on a stalled peer rank during commit, and in staged mode the maximum drain stall before watermark backpressure raises an error instead of hanging forever. Its real content is the guarantee: the engine fails loudly rather than hanging indefinitely. |
| `Aggregate` | `off` | `on`: node-aggregated co-write drain — one `data.<node>` sub-file per node co-written by its ranks at pre-assigned disjoint offsets (one store client per rank, one file per node). Store file count scales with nodes, not ranks. Must be uniform across ranks. Rationale and the two Lustre mechanisms it needed: [Semantics](semantics.md#aggregate). **Keep `off` on GPU nodes** (one rank per GPU): with few ranks, file count is never the problem, every rank is already an independent store client, and co-write coordination would be pure overhead — the split default is the fastest staging path there. It earns its keep on CPU-fat nodes (32–64 ranks/node). |
| `StagingSync` | `false` | `fsync` the staging file inside `EndStep`; durability is always `bbstage.final` on the final store. |

{: .param-names}

## What each default protects you from

| default | failure mode it prevents |
|---|---|
| `CloseMode=durable` | surprise semantics: Close() means it |
| watermark 1 GiB | unbounded backlog → tier exhaustion or OOM; exit-time cliff |
| `Aggregate=off` | co-write coordination cost when file count is not the problem |
| `CommitTimeoutSecs=300` | infinite hang on a dead peer rank or a wedged store |

{: .param-names}

## Full reference

The complete protocol semantics and limitations live in the fork's user
guide: [`docs/user_guide/source/engines/bbstage.rst`](https://github.com/denisbertini/ADIOS2)
alongside the engine source.
