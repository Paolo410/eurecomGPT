# Phase 2 — Reflection: where does parallelism stop helping?

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->883.29<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->3.107<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->464830.9<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->12763.8<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.5099<!--/answer--> |

(Threading plot: output of cell 12 in `phase-2-tiny-gpt/notebook.ipynb`.)

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2635.261 ms; 2 threads: 1977.5 ms; 4 threads: 2439.35 ms; 8 threads: 2910.995 ms. The fastest was 2 threads (1977.5 ms).
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Both versions do the same arithmetic, but the naive one runs it as 65,536 tiny Python-level steps (one 64-element dot product per query/key pair for T=256), so the interpreter's per-iteration overhead and the allocation of small objects dominate, and the CPU never works on large contiguous blocks. The vectorized version hands whole matrices to NumPy, which calls a compiled BLAS: the loops run in machine code, data is processed in cache-friendly tiles, and SIMD (AVX) instructions work on several floats per instruction. Result: 883.29 ms against 3.107 ms, about 284 times faster.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve stops improving almost immediately: going from 1 to 2 threads only gave 1.33x (2635.261 to 1977.5 ms), and with 4 threads (2439.35 ms) and 8 threads (2910.995 ms) it got worse, even slower than a single thread. My Colab runtime has only 2 vCPUs, so beyond 2 threads the extra threads share the same cores: they are time-sliced, wait for each other at the synchronisation point after every parallel operation, and compete for the same caches and memory bandwidth. Part of each step (Python overhead, optimizer, autograd bookkeeping) is also serial, so more threads cannot help it (Amdahl).
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A transformer training step is dominated by large dense matrix multiplications (attention and the MLP) applied to every token of the batch: the same operation on thousands of independent numbers, which a GPU runs on thousands of cores (SIMT) with very high memory bandwidth. With the model, data and step count fixed, 100 steps took 464830.9 ms on the CPU and 12763.8 ms on the GPU, about 36.4 times faster. A workload that would not benefit is one that is sequential or latency-bound, such as generating text one token at a time with a tiny batch, or reading files: the kernel-launch and host-to-device copy costs outweigh any parallel gain.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Each experiment hit a different kind of limit. Vectorization (883.29 to 3.107 ms, about 284x) met almost none: the work divides perfectly and the only cost was Python overhead, which BLAS and SIMD remove. Threads were bound by hardware I do not have and by coordination: Colab gives 2 vCPUs, so 2 threads was best (1977.5 ms), and at 8 threads (2910.995 ms) the cost of sharing 2 cores exceeded any gain, even losing to 1 thread (2635.261 ms). Even at 2 threads the speed-up was only 1.33x because part of the step cannot be divided (Amdahl). The GPU gave 36.4x (464830.9 vs 12763.8 ms), far from its raw core count: the batch is built on the CPU and copied to the device every step, the model is small, and memory bandwidth and kernel launches limit how much of the GPU I actually use. So parallelism stops helping when the serial fraction, the available cores, the synchronisation cost or the data movement becomes the bottleneck, whichever is reached first.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would sweep the batch size (for example 8, 16, 32, 64, 128 and 256) on both CPU and GPU and measure the time per sample. If the GPU speed-up keeps growing with batch size and then flattens, small batches were leaving the GPU idle; if it stays flat from the start, memory bandwidth or data movement is the limit instead, which would change my explanation of the 36x.
<!--/answer-->
