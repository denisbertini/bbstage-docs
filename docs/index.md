# bbstage

**bbstage** is a write-only [ADIOS2](https://github.com/ornladios/ADIOS2)
engine that decouples the application-visible checkpoint cost from the
(slow) final store: each step is spilled synchronously to fast node-local
media, and background workers drain it to the final store in large
sequential writes **while the application computes**. The output is
standard BP5 — read it back with the stock `bp5` engine, `bpls`, or any
BP5 reader.

![bbstage architecture: the checkpoint step spills to the node-local
staging tier at RAM/disk speed and returns; background drain lanes copy to
the shared filesystem, off the critical
path.](assets/bbstage-architecture.jpg){ loading=lazy }

!!! info "What it is — and is not"

      * **Is**: insurance against store contention. A quiet, headroomed store and bbstage tie at the application floor — there is no magic on a good day. Under contention, the engine absorbs the store's slowness into a background queue with an explicit, observable envelope.
      * **Is not**: a speedup over the page cache on idle stores, nor a filesystem. It adds one staging copy per checkpoint, always, and pays out in tails, metadata shape, and governed backpressure.

## Two close modes

| mode | `Close()` returns when… | durability on return |
|---|---|---|
| `durable` (default) | everything is drained and committed | all bytes on the final store |
| `staged` | the last step reached the node-local tier | only what was finalized; completion is marked by the **`bbstage.final`** file — never delete before it exists. See [Semantics](semantics.md) |

## Where the numbers come from

The whole story in one line — and one [main equation](tuning.md#equation):
a checkpoint costs `S/B` (spill ÷ tier speed) as long as demand stays
within drain capacity and backlog under the watermark; the store's speed
and jitter appear nowhere else in the application's wall time.

Every performance statement in these pages is tied to a measurement on real
Lustre systems (job IDs in [Tuning by measurement](tuning.md) and
[Methodology](methodology.md)). The headline constants, so you know what to
expect before reading the details:

| quantity | measured value | depends on |
|---|---|---|
| store supply | ≈ per-stream rate × stream count | **currency** (buffered vs `O_DIRECT`: ~25×), width, store |
| engine drain lane | 0.24–0.43 GiB/s per lane per store; ~3× faster on ~1 GiB files | store path, **file size** (per-file fixed cost) |
| staging tier | 1.9–5.5 GiB/s (NVMe class), ~8–50 GiB/s (`/dev/shm`) | node class — probe on arrival |
| benefit | 3.4× per-checkpoint under store contention; **1.0× on quiet days** | store contention, by design |

## Sibling projects & lineage

* **bblite** — the same absorb→drain architecture generalized below the
  framework, as a daemonless `LD_PRELOAD` shim at the POSIX boundary:
  [github.com/denisbertini/bblite](https://github.com/denisbertini/bblite)
* **BBView** (Koyama & Tatebe, REX-IO 2025) — the staging + background-drain
  lineage bbstage follows, implemented fully in-process. No view
  transformation: the staged payload is already serialized in final BP5
  layout, so background work is a byte copy plus a deferred metadata commit.

## Status & fork layout

Fork addition on a pinned upstream base: `master` = upstream + bbstage.
The engine is purely additive (`source/adios2/engine/bbstage/` plus
registration lines), so re-aligning with upstream is a small rebase.
Tests: `testing/adios2/engine/bbstage/` (with `ADIOS2_BUILD_TESTING=ON`).
Write-only engine; POSIX staging only; read side is stock `bp5`.

[:material-book-open-variant: Get started — Quickstart](quickstart.md)
