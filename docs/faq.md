# FAQ

**Does bbstage make checkpoints faster on an idle store?**
No — and it should not. On a quiet, headroomed store both the stock BP5
engine and bbstage tie at the application floor (the page cache absorbs
both). The engine's win is proportional to store contention: 3.4×
per-checkpoint under contention, 1.0× on calm days. Benchmark under
contention or not at all.

**Why does my store probe say 1 MiB/s while the engine writes 2 GiB/s?**
Different currencies and layers. The probe (`dd oflag=direct`) reads OST
digestion; the engine's buffered drain is admitted into cluster server RAM
and app-side numbers are spill-to-tier throughput in the first place.
Right after heavy writes, the pinned probe may be measuring **your own
job's writeback debt**. See [Diagnostics](diagnostics.md#store-probes-that-do-not-lie).

**Is staged mode safe with restarts?**
Key everything on `bbstage.final`: written when a checkpoint is complete;
a clean exit finalizes everything pending first. Before the marker exists,
a staged checkpoint may not survive a crash — same class of exposure as
losing a page cache, but stated and observable. [Semantics](semantics.md).

**How big should the watermark be?**
The default (1 GiB/rank) is measured-sane: bounded tier usage, small exit
tails, visible backpressure. Raising it banks backlog and moves repayment
to exit. Raise only if `drain_capacity × interval ≥ size × headroom` —
then it barely matters. [Tuning](tuning.md#watermark-sizing).

**Should I use `Aggregate=on`?**
When store *file count / MDS load* is your constraint (many ranks × many
files) — yes, with eyes open: it needs the leader-`ftruncate` and
chunked-close mechanisms (both built-in). **On GPU nodes: keep the split
default (`off`).** One MPI rank per GPU means only a handful of writers
per node — file count stays small, each rank is already its own store
client, and the co-write coordination buys nothing you need; the everyone-
writes default is the faster staging path. `Aggregate=on` is for CPU-fat
nodes (32–64 ranks/node) where the R×S file storm is real.
[Semantics](semantics.md#aggregate).

**Should `DrainStreams` be > 1?**
No, unless you can point at a per-*file* throughput throttle on your
store. Measured A/B in its own best regime: flat in k, ~20% writer tax,
variance up. [Tuning](tuning.md#drain-k-a-measured-demotion).

**Which staging tier should I use?**
The node's local disk (`/tmp`, NVMe) — the engine default, zero
configuration, graceful under pressure. It is the right answer on every
node class we measured; the specialists' chapter has the footnote-level
exceptions: [Tuning](tuning.md#staging-tier).

**My logs are unreadable interleaved garbage.**
Per-rank stderr: `2> logs/run.$SLURM_PROCID.err`. One shared stderr across
ranks is un-parseable confetti — every field you tried to keep got
interleaved with every other rank's.

**How do I know the drain kept up?**
`BBSTAGE_TIMING=1` → `drain-stats tail: ... 0 GiB` at exit = zero backlog,
zero exit bill. Nonzero tail = the exit payment, in one line; growing tails
across steps mean demand is outrunning drain — [Diagnostics](diagnostics.md#drain-stats).

**Does it need a daemon, privileged setup, or a reader change?**
No, no, and no: fully in-process, output is standard BP5 (stock `bp5` /
`bpls` read it), and the only artifact outside your files is the
`bbstage.final` marker.
