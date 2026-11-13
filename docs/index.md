# bbstage

**bbstage** is a write-only [ADIOS2](https://github.com/ornladios/ADIOS2) engine built to prevent slow cluster filesystems (such as Lustre or GPFS) from stalling simulations during checkpoints or diagnostics.

Instead of pausing your code while data travels across the network, **bbstage** writes each step immediately to node-local NVMe storage (`/tmp`). Background threads then push data to the permanent filesystem **while the simulation computes the next time step**. 

Outputs use the standard **BP5 format**, readable via `bp5`, `bpls`, Python, ParaView, or VisIt without modifying your simulation or analysis scripts.

![bbstage architecture: the simulation writes to fast node-local storage and immediately resumes computing, while background threads copy the data to the shared filesystem.](assets/bbstage-architecture.jpg){ loading=lazy }

!!! tip "When should you use bbstage?"

    * **Protect against I/O stalls:** Absorbs unpredictable slowdowns from busy shared filesystems in the background so your simulation runtime stays consistent.
    * **Behavior on quiet clusters:** When the shared filesystem is idle, performance matches standard BP5.

## Choosing how checkpoints finish (`CloseMode`)

| Mode | When does your code move on? | Data safety guarantee | Best used for... |
| :--- | :--- | :--- | :--- |
| **`durable`** *(default)* | Waits until all data finishes copying to the permanent filesystem. | **100% permanent.** Safe on the shared filesystem upon return. | **Final outputs** or critical checkpoints. |
| **`staged`** | Returns immediately once data reaches node-local storage. | **In-transit.** Background threads finish the copy asynchronously. | **Frequent mid-run checkpoints** for maximum simulation speed. |

!!! warning "Important rule for `staged` mode"
    Do not read, move, or delete the output directory until **`bbstage.final`** appears inside it, signaling all background transfers are complete (see [Semantics](semantics.md)).

## What kind of speedup can you expect?

Your I/O wait time depends on local storage speed rather than network congestion, costing roughly `S / B` per checkpoint — data size divided by local storage speed ([the balance equation](tuning.md#balance)).

| Storage Layer | Benchmarked Speed | Measured per... | Impact on Your Simulation |
| :--- | :--- | :--- | :--- |
| **Node-local NVMe (`/tmp`)** | **~1.9 GiB/s** (PCIe Gen3) to **~5.5 GiB/s** (PCIe Gen4); bursts hit memory speed via page cache. | **Node** (shared by all ranks; 4–8 ranks saturate the drive). | The speed your simulation experiences when writing checkpoints. |
| **Shared Cluster Filesystem** | **~0.24 – 0.43 GiB/s** (up to ~0.7 GiB/s for ~1 GiB files). | **Rank** (total node rate scales with rank count, up to the store's saturation ceiling). | Handled quietly in the background. |
| **Overall Checkpoint Time** | **Up to 3.4× faster** under heavy load (**1.0×** on idle clusters). | **Full job** (8 nodes × 32 ranks, 64 GiB: 7.7 s vs 26.5 s). | Eliminates multi-minute I/O hangs. |

* **Hardware variance:** Drive ceilings depend on NAND flash rather than slot generation; probe your specific node.
* **Asynchronous rates:** Local NVMe speed is a per-node ceiling, whereas filesystem speed is per-rank. They do not compete simultaneously.

The governing inequality for background drain capacity is:

$$
\underbrace{\frac{S}{\Delta t}}_{\text{checkpoint size} \div \text{interval}}
\;\leq\;
\underbrace{R \times v_{\text{rank}}}_{\text{ranks per node} \times \text{store rate per rank}}
$$

where $S$ is the checkpoint data written **per node** (all its ranks together) and $\Delta t$ is the time between checkpoints — so the left side is the average data rate your checkpoints demand, and the right side is the rate the background can deliver on that node.

Example: 8 ranks/node × 0.24 GiB/s ≈ **2 GiB/s of sustainable checkpoint demand per node**. - **Below this limit:** Shared filesystem slowdowns do not affect your runtime at all.
- **Above this limit:** bbstage pauses your simulation cleanly at a checkpoint boundary—never mid-calculation—until the background transfer catches up (see [Methodology](methodology.md)).
- **Two different clocks:** this limit is a long-run *average*, not the speed of your checkpoint write. The write itself is a burst at node-local NVMe speed (that is exactly what the fast local disk buys: a 16 GiB checkpoint pauses the run for ~3 s at 5.5 GiB/s, not the minutes the filesystem would demand); the drain then carries the burst away slowly over the compute time until the next checkpoint. The buffer is what turns fast bursts into a sustainable average.

## How it works with your code

* **Zero code changes:** Switch the engine name in ADIOS2 calls, config files, or middleware like **openPMD-api** (e.g., WarpX), as detailed in the [WarpX recipe](warpx.md).
* **No extra daemons:** Runs natively inside existing MPI processes.
* **Node aggregation (`Aggregate=on`):** Combines outputs on CPU nodes to reduce file-count pressure on shared filesystems.
* **GPU nodes:** Leave `Aggregate` off; single-rank-per-GPU layouts already run fastest with default parallel staging.

## Background & Related Tools

* **Origin (BBView):** Built on the burst-buffer concept of **BBView** — Koyama & Tatebe, *A View-Aware Burst-Buffer Mechanism for MPI-IO*, REX-IO @ IEEE Cluster 2025, [doi:10.1109/CLUSTERWorkshops65972.2025.11164194](https://doi.org/10.1109/CLUSTERWorkshops65972.2025.11164194) ([code](https://github.com/tsukuba-hpcs/bbview)); journal version, free access: [doi:10.1587/transinf.2026PAP0001](https://doi.org/10.1587/transinf.2026PAP0001). BBView buffers MPI-IO file views via a background daemon; **bbstage** adapts this absorb-then-drain design directly inside ADIOS2 in pre-serialized BP5 format, so no daemon is needed.
* **Sibling tool (`bblite`):** For legacy non-ADIOS2 codes, [bblite](https://github.com/denisbertini/bblite) provides similar POSIX background staging.

[:material-book-open-variant: Get started — Quickstart](quickstart.md)
