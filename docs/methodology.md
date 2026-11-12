# Methodology

These are the rules the campaign that produced every number on these pages
was run by. They are the transferable output — more portable than any
constant in [Tuning](tuning.md).

## 1. Measure supply in the engine's currency

The same store presented ~50 MiB/s to an O_DIRECT qd=1 probe and ~1.4
GiB/s+ to the engine's buffered, app-shaped traffic — at identical width,
same hour. Sizing a burst-buffer envelope (or any drain budget) from the
wrong currency misses by ~25×, usually *closed when open*. Pair every
supply claim with: **write class, stream count, queue depth, block size,
sustained vs burst.**

## 2. Probe on arrival — at every level

Constants did not survive relocation at *any* layer:

* per-lane drain rate: 1.8× between two stores;
* staging tier: 1.9 vs 5.5 GiB/s between node classes **in one cluster**,
  with opposite queue-depth behavior;
* SLC cliff: exists on consumer NVMe, absent on the enterprise drives;
* even "per-stream ceiling" turned out to depend on **file size**.

Rule: no constant is adopted across a site (or node class, or file size
change) without a 2-minute re-probe.

## 3. Burst and sustained are different numbers; label which you quote

Server RAM absorbed 75 GiB/s of "fsync'd" writes; the honest sustained
admission was 1–2 orders lower. Client page cache, server RAM, and the OST
layer are three different "store speeds" ([Tuning](tuning.md#the-three-absorption-layers)).
Every absorption layer has its own cliff — including the ones you did not
build.

## 4. Design every A/B so it can *lose*

`DrainStreams` was built, then the first A/B was shaped such that it
*could only* find "no difference" (per-rank demand under the assumed
ceiling). The redesign that could prove it right — fat ranks, demand above
the ceiling, drift control, store probes bracketing each arm — proved it
wrong instead. An experiment that cannot lose is a demo, not a test.

## 5. Audit the instrument before trusting the system

Three instruments lied before any system did:

* an 82 GiB/s "buffered store" reading = page cache, not store;
* `fio` buffered + `end_fsync` = 75 GiB/s — settled with a 20 GiB probe
  that `run=` *does* charge the flush, converting an "instrument defect"
  into a *finding* (server-RAM absorption, layer 2);
* a "12 GiB" checkpoint was a 2.4× arithmetic myth — the engine's own
  byte counters (drain-stats fragment sizes) proved 5 GiB.

Corollary: **believe the engine's byte counters** — and cross-check each
instrument against a second one before adoption (dd ↔ fio ↔ drain-stats
triangulated the same store three ways).

## 6. Make the queue observable or you do not own it

The product of a staging engine is not speed, it is **an envelope you can
state, watch, and bound**: watermark visible, drain-stats per rank,
`bbstage.final` as a durability marker, waits arriving at defined
boundaries instead of mid-kernel. The same async mechanism is weather
when the kernel runs it and climate control when you do.

## 7. Default = the old behavior; adoption needs measurement

Every engine feature ships opt-in with byte-identical default output
(k=1 verified byte-identical across k=1/4/16), and is promoted only by a
measured win. Two features (drain-k, staging preallocate on block tiers)
were *demoted by measurement* and remain dormant — documented with their
verdicts, so the next reader does not re-pay the tuition.

## 8. Logging discipline is an experiment-design problem

64 ranks sharing one stderr shred multi-write log lines beyond repair;
per-rank files ($SLURM_PROCID) were the difference between a paper figure
and archaeology. Related traps: stale background jobs (kill guards:
`pkill -f "fio/f[i]o"`), leftover handshake files (abort loudly on
stale state), and concurrent scheduler steps confusing MPI wireup.
