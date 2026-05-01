# A2 REFLECTION

> Complete every section. CI will:
>
> 1. Verify all `## Section` headers are present.
> 2. Verify each section has **at least 50 words**.
>
> No automatic content grading: your prose is read by a human, and the
> reasoning question at the end is marked on a 0 / 0.5 / 1 scale. Numbers you
> quote here do **not** have to match canonical re-run timings exactly —
> queue variance is real. CHOICE.md is graded against your own committed
> `perf-results-a2.json`, not against canonical times.

## Section 1 — Task decomposition

Describe your task decomposition for the tasks variant. Per-pixel? Per-tile? What tile size (or `grainsize`) did you use and why? Minimum 50 words.

For the tasks variant, I used a per-tile decomposition instead of spawning one task per pixel. The Mandelbrot workload is highly irregular near the set boundary, so using tiles reduces task creation overhead while still providing dynamic load balancing. Each TILE × TILE region was processed as one OpenMP task inside a `parallel` and `single` region. I used OpenMP task scheduling with a moderate grainsize to avoid generating too many very small tasks. This approach balances runtime overhead and scheduling flexibility more effectively than per-pixel task creation.

## Section 2 — Comparison: parallel_for vs tasks

Looking at the measured times in your `tables.csv`, at which thread counts did tasks win, and at which did parallel_for? What does this pattern tell you about the workload shape of this Mandelbrot region? Minimum 50 words.

According to the measured timings in `tables.csv`, the `parallel_for` implementation outperformed the tasks implementation at all tested thread counts, especially at 64 and 128 threads. The tasks implementation introduced substantial task management and synchronization overhead, which dominated the runtime on the Rome node. The `parallel_for` version using dynamic scheduling achieved much higher scalability and efficiency. This pattern suggests that although the Mandelbrot workload is irregular, the cost imbalance in this specific region was handled efficiently enough by dynamic scheduling without requiring expensive task creation.

## Section 3 — Memory model considerations

Did your task decomposition require any explicit synchronisation beyond what `taskwait` / `taskgroup` provide? Did you see (or rule out) any potential race in a shared accumulator? Minimum 50 words.

The task-based implementation required careful handling of shared state to avoid races in the accumulator variable. Initially, directly updating the shared `outside` variable inside tasks could lead to data races. To address this, synchronization mechanisms such as atomic updates or structured task synchronization with `taskgroup` were necessary. I also verified correctness by comparing outputs across multiple thread counts and repeated executions. The OpenMP memory model guarantees synchronization at the end of taskgroup and worksharing constructs, which helped ensure consistent results.

## Section 4 — Your recommendation

Your `CHOICE.md` picks one variant. Restate the recommendation here and summarise the *one* strongest piece of evidence for it (from your own measurements). If you picked the variant your own data showed as slower, your justification keyword (from the defensible-keyword enum) goes here. Minimum 50 words.

I recommend the `parallel_for` implementation for this workload. The strongest evidence comes from the measured performance on the Rome node at 128 threads. The `parallel_for` implementation achieved approximately 0.073 seconds, while the tasks implementation required about 2.876 seconds. This represents a very large performance gap in favor of the dynamically scheduled `parallel_for` approach. Although task parallelism can theoretically improve load balance for irregular workloads, in this implementation the scheduling and synchronization overhead outweighed the benefits, making `parallel_for` the clearly better choice.

## Reasoning question (instructor-marked, ≤100 words)

**In at most 100 words, explain *when* a task-parallel decomposition beats a parallel-for for kernels with this class of workload.**

Task-parallel decomposition outperforms parallel-for when the workload has highly irregular and unpredictable execution costs that cannot be balanced efficiently with standard loop scheduling. Tasks allow the runtime to dynamically redistribute work between threads as execution progresses. This is especially useful for recursive algorithms, adaptive meshes, graph traversal, or highly uneven Mandelbrot regions near complex boundaries. However, tasks only outperform parallel-for when the amount of work per task is large enough to amortize task creation and synchronization overhead.
