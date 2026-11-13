# Diagnostics

## `BBSTAGE_TIMING=1` — what each phase means

Per phase, per rank, on stderr. Each phase pins a *different* mechanism,
which is what makes them diagnostic:

| phase / column | mechanism it measures | if it is large |
|---|---|---|
| `watermark` | store-lag backpressure (writers waiting on the drain) | your demand exceeds drain capacity: check `drain-stats`, widen lanes/files, or accept the queue |
| `spill-alloc` | staging charge site | multi-second alloc stalls = the tier is refusing the load — lower the watermark or check the tier's free space |
| `spill-copy` | tier bandwidth site | on an XFS/ext4 tier: page-cache writeback throttling — see tier table in [Tuning](tuning.md#staging-tier) |
| `marshal` / `offsets` / `commit-enqueue` | serialization, bookkeeping, queue hand-off | usually app-side, not I/O |

## `drain-stats` — the engine's own throughput currency {: #drain-stats}

```
bbstage rank 17 drain-stats during-run: 6 jobs, 2.34 GiB in 9.9 s = 243 MiB/s
bbstage rank 17 drain-stats tail:       2 jobs, 1.00 GiB in 4.0 s = 256 MiB/s
```

* **`during-run`** = busy-time drain rate (sum G / sum S across ranks'
  files gives the arm's aggregate busy rate).
* **`tail`** = what the finalizer still had to do at exit — **the exit
  bill, in one line.** 0 GiB = the drain kept up all run (the goal).
* Compare these against your store probe **in the engine's currency** —
  not against `dd` (see the currency law in [Tuning](tuning.md#currency)).

## Reading a run: three quick checks

1. **Did the engine do anything?** `grep drain-stats logs/*.err` — no
   lines, wrong engine/env.
2. **Was the drain the constraint?** `during-run` rates pinned near the
   per-file rate (≈ `size / (0.24 s + size/910 MiB/s)` on the measured
   store) while `watermark` phases > 0 ⇒ store-bound; rates well below ⇒
   you are inside the envelope and the store is not your problem.
3. **Was the tier the constraint?** `spill-copy` large while drain idle ⇒
   tier-bound ([tier table](tuning.md#staging-tier)).

## Store probes that do not lie {: #probes}

```bash
# floor currency (O_DIRECT, qd1): reports the OST-digestion state,
# NOT what the engine will get:
dd if=/dev/zero of=$STORE/probe.$SLURM_JOB_ID.$SLURM_PROCID \
   bs=1M count=512 oflag=direct; rm -f ...

# engine currency: read drain-stats during-run from a short real run,
# or fio with --end_fsync=1 parsed as io=/run= (validate your fio charges
# the flush: one 20 GiB buffered run settles it).
```

Interpreting a *degraded* probe: right after heavy writes, O_DIRECT probes
stay pinned low while buffered admission still rides server RAM — often
because **your own job's writeback debt is still being digested**. Wait,
re-probe, believe time series not points.

## Operational gotchas (each bought with a broken run)

* **One stderr per rank.** `2> logs/$TAG.$SLURM_PROCID.err`. Shared
  stderr across 64 ranks interleaves multi-write lines into confetti —
  unrecoverable after the fact.
* **Heuristic greps count banners.** Counting "watermark" lines counts
  config banners too (identical counts across arms = tell). Use the phase
  timing numbers, not string matches.
* **Guard your build features.** Before an A/B, verify the library is
  current by grepping a marker literal from the feature's log line in
  `libadios2_core.so` — a silently-absent feature produces a confident,
  worthless experiment.
* **Bash idiom trap**: `local k=$1 tag="k$k"` expands `$k` from the
  *outer* scope. Two `local` statements. (This one mislabeled a control
  arm.)
