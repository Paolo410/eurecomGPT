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
The local run was by far the fastest at 0.017 s, then Spark at 3.7 s, and the cloud MapReduce was the slowest at 48.0 s. The fair comparison is local against cloud, since both do the same word count on the same 60 documents, and the cloud job took roughly 2,800 times longer. Counting words is trivial, so that time is overhead: 20 function calls (16 tasks plus 4 retries), many small reads and writes to Cloud Storage, the workflow scheduling and the waiting between retries. Spark did a heavier job on one machine in 3.7 s, so most of the cloud time is overhead.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job had 4 stages, so 3 shuffles. They come from the `reduceByKey` in `term_freq`, the `reduceByKey` in `doc_freq` and the `join`. In the chaos run there were 4 retries, which is 25% of the 16 tasks. With a 30% chance of failing on each attempt, a task fails about 0.4 times before it works, so I would expect around 7 retries. Four is a bit low, but one random run can easily land there.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
`parallelize` creates the first RDD. `flatMap`, `map`, `reduceByKey` (used twice) and `join` are transformations, so they are lazy and only record the next step. `collect` is the only action. When I ran `build_tfidf_rdd` it came back immediately and nothing had been computed, it had only built the lineage that `toDebugString` printed. Then `collect` made Spark split the job into 4 stages and 12 tasks and return 5,975 rows in 3.7 s.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Spark would only redo that one partition, using the shuffle files that stage 4 wrote as its input. If the dead worker also took those files with it, Spark would rerun just the stage 4 tasks that produced the missing pieces, starting again from the source documents. All the other partitions are reused, so the recovery starts at the first shuffle boundary, between stages 4 and 5.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows. The `mr-wordcount` workflow (`workflow.yaml`) starts the tasks, waits for the map phase and retries failures.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function `mr-worker` (`main.py`), which runs my `map_wc`, `shuffle` and `reduce_wc` for each task.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage, in the bucket `malugani-eurecomgpt-eurecomgpt`, which holds the input splits, the shuffle files and the output partitions.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The `partition()` function in `mapreduce.py`, which sends each word to reducer `crc32(word) % num_reducers`.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The `parallel` step of `map_phase`. It only finishes when all 12 map tasks are done, so `reduce_phase`, the next step, can't start before the last mapper finishes.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
The worker reads its input from Cloud Storage and writes output files with fixed names, replacing whatever was there. So running a task again after a failure or a timeout leaves exactly the same files, and a retry can't change the result. If `run_map` appended instead, a task that crashed halfway or ran twice would leave duplicate pairs. The reducers would count them twice and the totals would be too high compared to the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
The workflow execution finished as SUCCEEDED in 48.0 s. In the worker logs there are 4 requests that returned HTTP 503, the simulated crashes, all within about 4 seconds. The workflow retried them after a short wait and every task went through in the end. The answer was identical to the local reference, with 2,221 distinct words and `matches_local` true.
<!--/answer-->
