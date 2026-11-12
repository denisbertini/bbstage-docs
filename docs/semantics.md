# Semantics & durability

## What `Close()` guarantees

| mode | `Close()` returns when | after return |
|---|---|---|
| `durable` | all data drained, metadata committed | checkpoint complete on the final store |
| `staged` | the last step reached the node-local tier | drain + commit continue in the background; a process-level **finalizer** completes them; a clean exit finalizes every pending session before the process ends |

In `staged` mode the node-local tier holds only the **in-flight drain
backlog** — not the whole run — no matter how many steps the session has,
because draining starts at `Open()`. The finalizer publishes
`<checkpoint-dir>/bbstage.final` when the checkpoint is complete.

| boundary | guarantee |
|---|---|
| clean process exit | every staged session fully finalized (this is the commit barrier) |
| `bbstage.final` exists | checkpoint durable and readable by stock BP5 readers |
| crash before finalization | checkpoints staged-but-not-finalized are lost — like any page cache, and stated here instead of hidden |
| store crash mid-drain | atomic publish: readers see a complete checkpoint or none — never a half-published one |

!!! warning "Never delete before the marker"

      Restart/cleanup scripts must not delete staged checkpoints before
      `bbstage.final` exists. Key archiving and deletion policies on the
      marker, not on directory appearance.

## Backpressure contract (staged mode) {: #backpressure}

The engine's degradation model is explicit — under a store slowdown you
get, in order:

1. **Invisible absorption**: backlog grows in the tier; writers keep
   paying only the spill cost (measured: wall time flat across 4× demand
   change, two stores).
2. **Bounded, visible waits at `EndStep`**: past `StagingWatermarkMB`,
   writers wait for the drain — at a defined checkpoint boundary, never
   mid-kernel at an arbitrary instruction like page-cache writeback
   throttling.
3. **Exit bill**: whatever is still undrained at exit finalizes before
   process end — `backlog ÷ drain_capacity`, paid once, in the open.
4. **Error, not data loss**: watermark stall beyond `CommitTimeoutSecs`
   throws; ENOSPC requires you to have disabled the watermark.

## Aggregate co-write drain (`Aggregate=on`) {: #aggregate}

Motivation: with per-rank BP5 layout, R ranks × S files-per-step = R×S
data sub-files on the store — a metadata storm that grows with rank count.
`Aggregate=on` keeps the fast per-rank node-local staging (the app-visible
path is unchanged) and drains into one `data.<node>` sub-file per node,
**co-written by every rank on the node** at pre-assigned disjoint offsets:
one store client per rank, one file per node. Output stays stock-BP5
readable.

!!! info "Why co-write, not single-writer-per-node"
      A single drain client per node is capped at the store's single-client
      write rate (~0.7 GiB/s measured), which in staged mode throttles the
      app back to store speed and destroys the win. One client per rank at
      one file per node keeps drain parallelism at aggregate file counts —
      the same multi-client-single-file pattern BP5's own aggregator chains
      use.

Two Lustre-specific mechanisms this required (both measured and fixed —
they are why an earlier aggregate design drained at single-client speed):

1. **Shared-inode size lock**: 32 clients extending a fresh `data.<node>`
   each step serialize on the MDT's inode-size lock (classic multi-client
   append pathology; `lfs setstripe` does not help). Fix: the node leader
   `ftruncate`s the sub-file to the step end at `EndStep` — one MDT op per
   node per step — so drain writes land *below* EOF on disjoint ranges and
   parallelize.
2. **Close waiting on an in-flight copy**: staged `Close()` used to join
   the drain, so a store-capped drain made every checkpoint pay a full
   copy at Close (13.9 s measured on 256 ranks). Fix: Close aborts the
   in-flight copy at its next 4 MiB chunk and hands it to the finalizer,
   which re-drains it idempotently (same bytes, same offsets). Close cost
   is bounded by one chunk (measured: 1 ms).

## Limitations

* Write-only engine; the read side is the stock `bp5` engine.
* POSIX staging only (no MPI-IO staging of the tier).
* `Aggregate` must be uniform across ranks.
* Staged-mode durability is deferred by design — the consistency contract
  above is the product.
