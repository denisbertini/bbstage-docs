# Quickstart

## From application code (C++)

```cpp
adios2::IO io = adios.DeclareIO("chk");
io.SetEngine("bbstage");
io.SetParameters({{"BurstBufferPath", "/tmp"},   // node-local staging dir
                  {"CloseMode", "staged"}});     // or "durable" (default)
adios2::Engine e = io.Open("chk.bp", adios2::Mode::Write);
// ... BeginStep / Put / EndStep ...
e.Close();
```

## From middleware you cannot reconfigure

When engines are reached through middleware that cannot pass engine
parameters (e.g. WarpX checkpoints via openPMD-api), environment variables
act as fallbacks — precedence: engine parameter > environment > default:

| Environment | Parameter |
|---|---|
| `BBSTAGE_CLOSE_MODE` (`durable`/`staged`) | `CloseMode` |
| `BBSTAGE_STAGING_WATERMARK_MB` | `StagingWatermarkMB` |
| `BBSTAGE_DRAIN_STREAMS` | `DrainStreams` |
| `BBSTAGE_AGGREGATE` (`on`/`off`) | `Aggregate` |
| `BBSTAGE_STAGING_PREALLOCATE` (`on`/`off`) | `StagingPreallocate` |
| `BBSTAGE_TIMING` (`1`) | — per-phase timings + drain-stats on stderr |

## A recommended staging-mode recipe

**Default tier: `/tmp` (node-local disk / page cache) — the engine
default, and the right choice almost everywhere.** It needs no
configuration: spill rides the page cache (which, for small backlogs,
behaves like RAM), the watermark bounds the backlog, and the local disk
absorbs whatever spills over — degrading *gracefully* instead of fatally.

```bash
BBSTAGE_TIMING=1 srun ... ./your_app      # per-rank timing + drain-stats on stderr
```

No other configuration is intended for normal use. See
[Tuning](tuning.md#staging-tier) for what the tier does under load.

## Reading the output

The output is standard BP5, published when finalization completes:

```bash
bpls chk.bp/                 # stock reader, no bbstage needed
test -f chk.bp/bbstage.final # ← durability/visibility marker (staged mode)
```

!!! warning "The `bbstage.final` rule"

      In `staged` mode, a checkpoint directory is *not durable* until
      `bbstage.final` appears inside it. Readers, archivers, and
      **delete-on-restart policies must key on that marker.** A clean
      process exit finalizes every pending session first; a crash loses
      checkpoints staged but not yet finalized — identical in kind to
      losing the page cache of a stock engine.

## First sanity check: is the engine doing anything?

`BBSTAGE_TIMING=1` prints, per rank and to **stderr**:

```
bbstage rank 17 drain-stats during-run: 6 jobs, 2.34 GiB in 9.9 s = 243 MiB/s
bbstage rank 17 drain-stats tail:       2 jobs, 1.00 GiB in 4.0 s = 256 MiB/s
```

`tail: 0 GiB` at exit means the drain kept up all run — the ideal
outcome. Growing tails or watermark waits point to the
[diagnostics guide](diagnostics.md).

!!! tip "One stderr file per rank"

      At ≥8 ranks, redirect per-rank stderr (`... 2> logs/run.$SLURM_PROCID.err`).
      A shared stderr across ranks interleaves multi-write log lines into
      unreadable confetti — learned the hard way.
