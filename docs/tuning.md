# Tuning by measurement

!!! abstract "This page's rule"
      Every number below is from a named measurement (jobs on two Lustre
      deployments, 2026). Constants here are **site facts, not universal
      laws** — the transferable part is the procedure: [Methodology](methodology.md).

## The three absorption layers

Anything that "absorbs writes fast" does it at one of three layers — and
each has its own speed, cliff, and bill:

| layer | who | burst | honest sustained | bill |
|---|---|---|---|---|
| 1 | client page cache | ~20 GiB/s | device | kernel writeback throttling, at an arbitrary moment, mid-step |
| 2 | **cluster server RAM** (Lustre `fdatasync` completes here) | ~75 GiB/s aggregate (probe j13590 validated the accounting) | OST-backed | deferred OST digestion; invisible |
| 3 | OST backing store | — | ~1.4 GiB/s direct@256 | this *is* the bill |

bbstage replaces layer-1 governance (kernel decides) with an **explicit
tier + watermark the operator sizes and observes** ([Semantics](semantics.md#backpressure)).

## Store supply is speed(currency × width) {: #currency}

Same store, same streams, same hour:

| instrument (currency) | 64 streams | 256 streams |
|---|---|---|
| `dd oflag=direct` qd=1 (floor) | ~50 MiB/s | ~1.4 GiB/s |
| engine drain (buffered, app-shaped) | **~15–27 GiB/s capability** (lanes) | — |

**A `dd`-sized envelope is ~25× too small — the classic self-inflicted
envelope.** And on a degraded store the gap goes the other direction too:
while the O_DIRECT probe read 1–6 MiB/s, the engine's buffered drain moved
236 GiB in 45 min (job j13606) — hundreds of ×, mediated by layer 2.

Per-store shapes differ as much as currencies: one deployment scaled
near-linearly with writer count (~5–6 MiB/s/stream × width); an
archive-class store saturated at ~640 MiB/s from 256 streams up. **Measure
your store with the engine's currency and your engine's width.**

## The drain lane constant is not a constant

Per-file drain throughput fits

```
t_file ≈ 0.24 s (per-file fixed cost) + size / ~910 MiB/s
```

(2-point fit: 80 MiB files measured 0.24 GiB/s/lane on store A,
0.43 on store B; 944 MiB files measured ~738 MiB/s — job j13606).
Consequences:

* at small files, the "lane wall" is mostly **per-file fixed cost**
  (open/flush/close under store load) → prefer **fewer, bigger files**:
  this is the strongest argument for `Aggregate=on` on MDS-loaded stores;
* the per-lane rate is a **store-path property** (1.8× between the two
  stores measured) — probe it on arrival;
* lane rate is **invariant to offered demand** — in-envelope, wall time
  per checkpoint = `Σ file_size / lane_rate` + finalize cost, flat vs
  demand (4× range, both stores, job-proven).

## Drain-k: a measured demotion

`DrainStreams` (k concurrent range-writes per slice) was built for the
"per-stream ceiling" story, then tested in the regime that story predicts
(1 rank/node, ~944 MiB/rank checkpoints, demand above the small-file
lane rate, job j13606):

| arm | drain rate | writer wall | tail at exit |
|---|---|---|---|
| k=1 | 738 MiB/s | 1.7–1.8 s/chkpt | 0 |
| k=4 | 725 MiB/s | 2.3 s/chkpt (+25%) | 0 |
| k=8 | 773 MiB/s | 2.0 s/chkpt, 4× variance | 0 |

Flat in k, taxes the writer: **keep `DrainStreams=1`.** One large
sequential stream per file already amortizes the fixed cost — the k knob
remains only for stores with genuine per-file throttles (single-OST
targets, per-stream client caps), absent from everything measured.

## The staging tier: per node class, certified ceilings {: #staging-tier}

| tier / node | sustained | width scaling | notes |
|---|---|---|---|
| `/dev/shm` (tmpfs) | ~8 GiB/s plain, ~50 with `StagingPreallocate=on` | — | pinned RAM; budget: `(watermark+backlog)×ranks + app` |
| Gen3 enterprise NVMe (Intel P4510 class) | **1.9 GiB/s** | none (1≡16 streams, 1M≡4M — NAND-bound) | O_DIRECT ≡ buffered; async I/O cannot help |
| Gen4 enterprise NVMe (Kingston DC class) | **5.5 GiB/s** | qd1 issue-limited (3.3); 4 streams ≡ 16; bs=4M alone ≈ ceiling | the engine's 8 ranks/node are already 8 streams — at the ceiling without any code change |
| consumer DRAM-less NVMe | 2.1 → 0.35 GiB/s after a few GiB | — | **SLC cliff**: a second checkpoint can land on it; gate NVMe staging on a sustained-write probe |

Two node classes in the *same cluster* differed 2.9× with opposite
width-scaling behavior — tier constants are per hardware, not per site.
The engine's natural spill shape (per-rank concurrent files) is the queue
depth; no O_DIRECT/io_uring/SPDK recipe beat the NAND ceiling in any test.

## When bbstage wins (and when it should not)

* **Field validation under contention** (8×32 ranks, 64 GiB/checkpoint,
  both engines minutes apart, store managing 2.5 GiB/s native): staged +
  aggregate ran **3.4× faster per checkpoint** (7.7 vs 26.5 s) and 2.8× on
  total job wall, draining ~1.2 GiB/s/node in the background while the app
  stayed at its compute floor.
* **Quiet headroomed days: both engines tie at the application floor.**
  That is the correct result — the win is proportional to contention. Do
  not A/B on a good day and conclude the engine is dead weight; do not
  benchmark on a bad store and conclude it is free.

## Watermark sizing

Default 1 GiB/rank is deliberate: it self-throttles the app when the store
cannot keep up and keeps tier usage and the exit tail small. Raising it
(e.g. 64 GiB) banks the backlog locally instead — the tier must then hold
`watermark × ranks/node` of slack, and the repayment shifts to process
exit (bigger `tail`). Raise it only when your measured
`drain_capacity × checkpoint_interval ≥ checkpoint_size × (1 + headroom)`
says the backlog is transient anyway — in which case it barely matters.
