# Phase 3 — Comparison Writeup

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->48.0<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.017<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.7<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->4<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
From fastest to slowest: the local run (0.017 s), Spark (3.7 s) and the cloud MapReduce (48.0 s). Only the local run and the cloud job are like-for-like: the same word count on the same 60 documents, yet the cloud one is about 2,800 times slower. The counting itself is trivial, so the time goes on overhead: 20 function calls (16 tasks plus 4 retries), many small Cloud Storage reads and writes, workflow scheduling and, in this run, retry waits. Spark did a heavier job in 3.7 s on one machine, which shows that the cloud job's time is mostly overhead, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job had 4 stages, so 3 shuffles: the `reduceByKey` in `term_freq`, the `reduceByKey` in `doc_freq`, and the `join`. In the chaos run there were 4 retries, about 25% of the 16 tasks. With a 30% failure chance per attempt, a task fails about 0.4 times before it succeeds, so around 7 retries were expected; 4 is a bit low but normal for a single random run.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
`parallelize` creates the first RDD. `flatMap`, `map`, `reduceByKey` (used twice) and `join` are transformations: they are lazy and only record how to build the next RDD. `collect` is the only action. When I ran `build_tfidf_rdd` it returned at once and nothing was computed; it only built the lineage, which I printed with `toDebugString`. `collect` then made Spark cut the job into 4 stages and run 12 tasks, returning 5,975 rows in 3.7 s.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Spark would recompute only that partition: the reduce-side task of the first `reduceByKey`, reading the shuffle files that stage 4 wrote. If the dead worker also held those files, it would re-run just the stage-4 map tasks that wrote the missing pieces, from the source documents. Everything else is reused, so recovery starts at the first shuffle boundary, between stages 4 and 5.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the `mr-wordcount` workflow (`workflow.yaml`) fans out the tasks, waits for the map phase and retries failures.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function `mr-worker` (`main.py`), which runs my `map_wc`, `shuffle` and `reduce_wc` once per task.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage, bucket `malugani-eurecomgpt-eurecomgpt`: the input splits, the shuffle files and the output partitions.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The `partition()` function in `mapreduce.py`: `crc32(word) % num_reducers`.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The `parallel` step of `map_phase`. It only finishes when all of its branches, the 12 map tasks, have finished, so `reduce_phase`, the next step, cannot start before the last mapper is done.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
A task reads its input from Cloud Storage and writes output files with fixed names, replacing what is there. Running it a second time, after a failure or a timeout, leaves exactly the same files, so a retry cannot change the result. If `run_map` appended instead, a task that crashed halfway or ran twice would leave duplicate pairs, the reducers would count them twice, and the word counts would come out too high and differ from the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
The workflow execution ended as SUCCEEDED in 48.0 s. The worker logs show 4 requests answered with HTTP 503, the simulated crashes, within about 4 seconds; the workflow retried them with its backoff and every task eventually succeeded. The result was still identical to the local reference: 2,221 distinct words, `matches_local` true. The retries cost time but not correctness.
<!--/answer-->
