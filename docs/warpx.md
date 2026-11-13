# Using bbstage with WarpX

WarpX writes checkpoints and openPMD diagnostics through
**openPMD-api → ADIOS2** — the application never calls `SetParameters`, so
the integration has two hooks:

1. **select the engine** via an openPMD config JSON (`engine.type`), and
2. **tune it** via the `BBSTAGE_*` environment fallbacks, which exist for
   exactly this middleware path ([Parameters](parameters.md)).

!!! warning "Validate before trusting"

      This page is a reasoned recipe, not yet a WarpX-validated run —
      your WarpX tests are the validation. Work through the
      [verification checklist](#verification-checklist) on a small job
      first: each layer (library → engine selection → drain → markers)
      has a one-line proof.

## The three moving parts

![WarpX integration: diag1.format and diag1.openpmd_config feed the
openPMD-api ADIOS2 backend and its engine.type JSON, which selects the
bbstage engine in the fork build; BBSTAGE_* environment variables tune
it.](assets/warpx-integration.jpg){ loading=lazy }

* The **fork's `libadios2`** must be the one openPMD-api loads — this is
  the single most common failure: WarpX runs, writes plain BP5, and
  nothing was staged, because the container's own ADIOS2 won the linker
  race. The guard below prevents it.
* `engine.type = "bbstage"` requires an openPMD-api new enough to pass
  the engine name through (0.14+); older versions hard-select BP5 — the
  checklist catches this in one line.

## Job submission script, annotated

```bash
#!/bin/bash
#SBATCH --job-name=warp-bb
#SBATCH --nodes=8
#SBATCH --ntasks-per-node=32
#SBATCH --partition=long
#SBATCH --time=4:00:00
#SBATCH --output=warp-bb_%j.out

set -euo pipefail

# ── 0. paths ────────────────────────────────────────────────────────────
CONT=/lustre/rz/dbertini/io-tests/adios-tests/container/plasma_pp.sif
FORK=/lustre/rz/dbertini/io-tests/adios-tests/ADIOS2   # bbstage fork build
export APPTAINER_BINDPATH=/lustre,/idril,/home
export APPTAINER_CONFIGDIR=/tmp/$USER

# ── 1. bbstage configuration (env fallbacks: middleware cannot pass
#       engine parameters; parameter > env > default) ─────────────────────
export BBSTAGE_CLOSE_MODE=staged        # start with "durable" (below) to
                                        # validate semantics before graduating
export BBSTAGE_STAGING_PREALLOCATE=on   # ~6x faster tmpfs spill, fail-fast on
                                        # RAM overrun (see budget, section below)
export BBSTAGE_AGGREGATE=on             # CPU runs with many ranks/node:
                                        # WarpX diags = one file per step per
                                        # quantity: the file-count case where
                                        # data.<node> co-write earns its keep.
                                        # GPU nodes (one rank per GPU): keep
                                        # the split default (off) — few ranks,
                                        # no file storm, per-rank writes are
                                        # the fastest staging path.
export BBSTAGE_STAGING_WATERMARK_MB=1024  # measured-sane default; see Tuning
export BBSTAGE_TIMING=1                 # drain-stats + phase timings on stderr

# ── 2. guards: two one-liners that prevent the two classic silent failures ─
srun --export=ALL bash -c "
  grep -qa 'bbstage rank' $FORK/build/lib64/libadios2_core.so.* \
    || { echo 'ABORT: this libadios2 has no bbstage'; exit 1; }
  a=\$(df -BG --output=avail /dev/shm | tail -1 | tr -dc 0-9)
  [ \"\$a\" -ge 30 ] || { echo \"ABORT: /dev/shm only \${a}G\"; exit 1; }"

# ── 3. openPMD engine selection (the engine.type passthrough) ────────────
cat > $SLURM_SUBMIT_DIR/bbstage.json <<'EOF'
{ "adios2": { "engine": { "type": "bbstage" } } }
EOF

# WarpX inputs (the relevant lines):
#   diag1.diagnostic_type = openPMD
#   diag1.format          = adios2
#   diag1.openpmd_config  = /path/to/bbstage.json
#   diag1.checkpoint      = 1           # restart files take the same engine

# ── 4. run: the fork's library FIRST on LD_LIBRARY_PATH ──────────────────
srun --export=ALL apptainer exec $CONT bash -c '
  export LD_LIBRARY_PATH='"$FORK"'/build/lib64:$LD_LIBRARY_PATH
  stdbuf -oL -eL warpx3d inputs_warpx \
    2> logs/warp-$SLURM_JOB_ID.$SLURM_PROCID.err'   # ONE stderr per rank
```

*The per-rank stderr is not style: 64 ranks on one stderr shred the
`bbstage`/`drain-stats` lines into confetti ([diagnostics](diagnostics.md)).*

## Staging budget (tmpfs recipe)

`/dev/shm` is pinned RAM. The job's memory allocation must cover:

```
(watermark + backlog) × ranks_per_node + WarpX footprint
```

With the default 1 GiB watermark × 32 ranks = **32 GiB of staging headroom**
— check this against your `--mem-per-cpu`, or lower the watermark, or use
`/tmp` (NVMe tier; degrades gracefully where tmpfs would OOM). With
`StagingPreallocate=on`, overruns fail fast with an explained `fallocate`
error instead of an OOM mid-run.

## Durability and restarts (the WarpX-specific rule)

WarpX restart reads from the **final store**. In `staged` mode a
checkpoint is complete only when `<diag>/.../<file>.bp/bbstage.final`
exists. Clean `warpx` exits finalize everything first — a **job kill or
crash** can lose the most recent, still-staged checkpoint. So:

* start colleagues on **`durable`**: every checkpoint is store-durable at
  close, semantics identical to plain ADIOS2, and you still gain the
  drain-path benefits (large sequential writes, co-write file shape);
* graduate to **`staged`** once the marker logic is understood — and if a
  script deletes old checkpoints on restart, **key the deletion on
  `bbstage.final`**, never on directory presence.

## Verification checklist

Run small (2 nodes, 10 steps), then grep the per-rank logs:

| # | check | command / expected |
|---|---|---|
| 1 | fork library loaded | log contains `bbstage rank 0 drain streams = 1` — the engine self-identifies at startup |
| 2 | engine selection took | **all** ranks log it, one line per rank; zero lines ⇒ openPMD still used BP5 (engine.type not passed ⇒ old openPMD-api) |
| 3 | drain is real | `grep drain-stats logs/*.err` shows `during-run` lines with plausible MiB/s |
| 4 | output is stock BP5 | `bpls <diag>/<file>.bp` works; aggregate mode shows exactly one `data.<node>` per node |
| 5 | markers exist | every completed checkpoint dir contains `bbstage.final` |
| 6 | WarpX restarts | `bpls`-check then a WarpX restart from that checkpoint — the round-trip is the actual claim |
| 7 | the win, honestly | compare step/diag times native (`engine.type: BP5`) vs bbstage **under store contention**; on a quiet store they should tie (that is the correct result, not a failure) |

If any check fails, the [diagnostics guide](diagnostics.md) names the
phase and layer to look at.
