# Parameters

Set via `io.SetParameters({...})`; the `BBSTAGE_*` environment fallbacks
(see [Quickstart](quickstart.md)) exist for middleware paths. Precedence:
**parameter > environment > default**.

| Parameter | Default | Meaning |
|---|---|---|
| `BurstBufferPath` | `$TMPDIR` or `/tmp` | Node-local staging directory. Point it at the node's fast local media; `/dev/shm` for RAM staging with a sufficient job memory allocation. |
| `CloseMode` | `durable` | `durable` or `staged`. Determines what `Close()` waits for — see [Semantics](semantics.md). |
| `StagingWatermarkMB` | `1024` | Staged mode: cap on staged-but-undrained bytes **per rank**. `EndStep` blocks above it until the concurrent drain catches up; after `CommitTimeoutSecs` without drain progress it throws (0 = unlimited). The engine's backpressure valve: bounded, visible waits instead of unbounded growth. |
| `DrainThreads` | `1` | Drain worker threads per rank (started from `Open` in both close modes). The node's drain width = threads × ranks-per-node, bounded by store concurrency scaling. |
| `DrainStreams` | `1` | Concurrent store streams per rank (max 16): the drain copies each staged slice with k disjoint range-writes; byte-identical output at any k. **Measured demotion: keep 1** — a fat-rank A/B found throughput flat in k (±7%) while writers paid ~20% wall and 2–4× jitter; one large sequential stream already amortizes the per-file fixed cost. For per-file-throttled stores only. Details: [Tuning](tuning.md#drain-k-a-measured-demotion). |
| `CommitTimeoutSecs` | `300` | Give up waiting on a stuck peer rank after this long (0 = forever); in staged mode also the stall timeout of watermark backpressure. |
| `Aggregate` | `off` | `on`: node-aggregated co-write drain — one `data.<node>` sub-file per node co-written by its ranks at pre-assigned disjoint offsets (one store client per rank, one file per node). Store file count scales with nodes, not ranks. Must be uniform across ranks. Rationale and the two Lustre mechanisms it needed: [Semantics](semantics.md#aggregate). **Keep `off` on GPU nodes** (one rank per GPU): with few ranks, file count is never the problem, every rank is already an independent store client, and co-write coordination would be pure overhead — the split default is the fastest staging path there. It earns its keep on CPU-fat nodes (32–64 ranks/node). |
| `StagingPreallocate` | `off` | `on`: `fallocate` each staging file at `EndStep` — **only on tmpfs tiers** (~6× faster spill there; RAM overruns fail fast before partial writes). A measured no-op on block-filesystem tiers (warned, ignored). |
| `StagingSync` | `false` | `fsync` the staging file inside `EndStep`. Meaningless on tmpfs (no device to flush); durability is always `bbstage.final` on the final store. |

## What each default protects you from

| default | failure mode it prevents |
|---|---|
| `CloseMode=durable` | surprise semantics: Close() means it |
| watermark 1 GiB | unbounded backlog → /tmp exhaustion or OOM; exit-time cliff |
| `DrainStreams=1` | writer tax from stream fan-out that scales nothing |
| `Aggregate=off` | co-write coordination cost when file count is not the problem |
| `StagingPreallocate=off` | behavior change on non-tmpfs tiers (no-op anyway) |

## Full reference

The complete protocol semantics and limitations live in the fork's user
guide: [`docs/user_guide/source/engines/bbstage.rst`](https://github.com/denisbertini/ADIOS2)
alongside the engine source.
