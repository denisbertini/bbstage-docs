# bbstage

**bbstage** is a write-only [ADIOS2](https://github.com/ornladios/ADIOS2) engine designed to prevent slow cluster filesystems (like Lustre or GPFS) from stalling your simulation during checkpointing or diagnostics output.

Instead of pausing your physics code while data is written across the network to the shared filesystem, **bbstage** saves each step immediately to ultra-fast local storage on the compute node (such as `/dev/shm` RAM disk or local NVMe SSD). Background threads then copy your data to the permanent filesystem **while your simulation continues computing the next time step**.

Your output is saved in standard **BP5 format**. You do not need to change your simulation code or your analysis scripts—you can read the files back using the standard `bp5` engine, `bpls`, Python, ParaView, or VisIt.

![bbstage architecture: the simulation writes to fast node-local storage and immediately resumes computing, while background threads copy the data to the shared filesystem.](assets/bbstage-architecture.jpg){ loading=lazy }

!!! tip "When should you use bbstage?"

    * **Use it to protect against I/O stalls:** Shared cluster filesystems fluctuate in speed depending on what other jobs are running. When the filesystem slows down, **bbstage** absorbs the delay in the background so your simulation wall-time stays predictable.
    * **What to expect on a quiet cluster:** If the shared filesystem is completely idle and fast, **bbstage** will perform roughly the same as standard BP5. Its main advantage is eliminating unpredictable, multi-minute I/O hangs during busy cluster periods.

## Choosing how checkpoints finish (`CloseMode`)

When your simulation finishes writing a checkpoint and closes the file, you can choose between two safety modes depending on your workflow:

| Mode | When does your code move on? | Data safety guarantee | Best used for... |
| :--- | :--- | :--- | :--- |
| **`durable`** *(default)* | Waits until all data has finished copying to the permanent filesystem. | **100% permanent.** Data is completely safe on the shared filesystem when `Close()` returns. | **Final outputs** or critical checkpoints before a job ends. |
| **`staged`** | Returns immediately as soon as data reaches the compute node's local storage. | **In-transit.** Background threads are still copying data to the permanent filesystem while your simulation runs. | **Frequent mid-run checkpoints** where maximum simulation speed is the priority. |

!!! warning "Important rule for `staged` mode"
    If you use `staged` mode, **do not read, move, or delete** the output directory until a file named **`bbstage.final`** appears inside it. That marker file guarantees all background transfers have finished and your data is safely on the permanent filesystem. See [Semantics](semantics.md) for details.

## What kind of speedup can you expect?

With **bbstage**, the time your simulation spends waiting on I/O depends only on the speed of your compute node's local storage—not the shared cluster filesystem. As long as your checkpoints are spaced out enough for the background threads to keep up, network filesystem slowdowns will not affect your simulation runtime. (In measured terms this is `S / B` — size divided by local storage speed — [the main equation](tuning.md#equation) of the tuning chapter.)

Based on benchmarks on production Lustre clusters (see [Tuning by measurement](tuning.md)):

| Storage Layer | Speed (as benchmarked here) | Impact on Your Simulation |
| :--- | :--- | :--- |
| **Node RAM (`/dev/shm`)** | ~8 – 50 GiB/s | Your simulation sees this speed when writing checkpoints. |
| **Node Local NVMe SSD** | ~1.9 – 5.5 GiB/s | Your simulation sees this speed when writing checkpoints. |
| **Shared Cluster Filesystem** | ~0.24 – 0.43 GiB/s per stream | Handled quietly in the background while your physics solver runs. |
| **Overall Checkpoint Time** | **Up to 3.4× faster** under heavy cluster load | Eliminates long I/O stalls during busy cluster hours (**1.0×** when the cluster is empty). |

*(Note: the filesystem numbers above are facts about the clusters we benchmarked, not universal constants — probe your own cluster the same way, see [Methodology](methodology.md). And if you stage data in `/dev/shm`, the temporary files count toward your job's RAM limit. See [Quickstart](quickstart.md) for guidance on setting memory limits safely.)*

## How it works with your code

* **Zero code changes:** If your simulation already uses ADIOS2 (or middleware like **openPMD-api**, used by WarpX and PIConGPU), you can enable **bbstage** by changing the engine name — in your own ADIOS2 calls or XML/JSON configuration, or, for openPMD-based codes, in the middleware's config file (see the [WarpX recipe](warpx.md)). Tuning parameters can also travel as environment variables.
* **No extra daemons to launch:** Everything runs automatically inside your MPI job's existing processes.
* **Node aggregation (`Aggregate=on`):** For large runs using many MPI ranks per node, **bbstage** can automatically combine outputs on each node to avoid overwhelming the shared filesystem with thousands of small files.

## Background & Related Tools

* **Origin:** **bbstage** builds on the burst-buffer staging concept introduced by *BBView* (Koyama & Tatebe, REX-IO 2025), re-engineered directly inside ADIOS2 so data is already formatted in BP5 layout before staging.
* **bblite:** If you have legacy codes that do not use ADIOS2, check out [bblite](https://github.com/denisbertini/bblite), a sibling tool that brings similar background staging to standard POSIX file writes.

[:material-book-open-variant: Get started — Quickstart](quickstart.md)
