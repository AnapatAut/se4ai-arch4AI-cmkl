# Week 6 worksheet: serving patterns

**Name:** Anapat Bunyapitaktamrong

**Student ID:** 01326172

This is **individual work**: each student completes and submits their own copy, in their own fork.

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in English.

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 492 | 587 | 93.5% | 100% |
| t0-small | 8 | 18 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

> The large model raises accuracy from 72.5% to 93.5%, but p95 latency rises around 30 times (18 -> 587 ms) and every review becomes a paid large-model call (100% vs 0%).

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 34.4 | 244 | 403 | 26 | 0 | 93.7% |
| t1-after | 9.6 | 417 | 1004 | 0 | 20 | 92.8% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

> p50 leaves out the 26 failed requests, so it only measures the successful ones. The errors column (26 vs 0) tells the real story, backed by the run time (34.4 s vs 10.0 s).

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

> A slow request frees its slot quickly, but a hung one holds its connection for 20 s. Hung requests pile up and block healthy ones too (run time 34.4 s vs 10.0 s, 26 failed).

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

> Queue pending reviews and label them later with the large model in an off-peak batch job (batch processing / asynchronous topology).

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

> At 0.1 s, even healthy calls (about 200 ms) time out, so nearly every review becomes pending. At 10 s, hung calls hold their slots 10 times longer, breaking QR2 and blocking healthy requests.

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 12 | 536 | 91.5% | 38% |
| t2-0.7 | 0.7 | 368 | 569 | 93.0% | 53% |
| t2-0.8 | 0.8 | 450 | 570 | 93.5% | 64% |
| t2-0.9 | 0.9 | 478 | 593 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

> 0.7: it keeps a safe margin above QR3 (93.0% vs 91.5% at 0.6), while 0.8 and 0.9 add many large calls for at most 0.5 points more accuracy. I give up cost (53% vs 38% large calls) and p50 latency (368 vs 12 ms).

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

> p95 is always a slow large-model call, which the cascade cannot speed up (587 -> 569 ms). p50 drops (492 -> 368 ms) because fewer large calls queue, and at 0.6 the small model answers most reviews (12 ms).

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

> A negated review ("Honestly the salad is not cold.") gets a confident negative from the small model (0.70), so it is never escalated. The owner sees a happy customer as unhappy.

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 4 | 539 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

> Short common reviews repeated word for word ("Delicious!", "Too salty") benefit, giving 35% hits. Unique reviews never hit, since a text's first appearance is always a miss.

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

> Invalidate it, or it serves the old model's stale answers. Include the model version in the cache key, flush on deploy, and add a TTL as a safety net.

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

> p50 does not change smoothly with the threshold: it jumped from 12 ms to 368 ms once more than half of the reviews went to the large model (0.6 -> 0.7).
