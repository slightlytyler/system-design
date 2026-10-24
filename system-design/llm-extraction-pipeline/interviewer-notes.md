# Interviewer notes: LLM extraction pipeline

Spoilers. Read these only after attempting the problem, or have someone play the interviewer with them.

## Answers to clarifying questions

Give these only when asked.

| Question                        | Answer                                                                                                                                 |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| How many users?                 | ~50k weekly active users. At peak, ~500 extraction jobs running at once.                                                               |
| How big is a job?               | Typically 50–200 papers × 3–10 columns. The largest are 1,000 papers × 20 columns (20k cells).                                         |
| How long are papers?            | Full text averages ~10k tokens; long papers exceed 50k. Some only have an abstract.                                                    |
| LLM latency?                    | p50 ~8s, p99 ~60s. Long papers are slower. Streaming is supported.                                                                     |
| LLM error rate?                 | 2–5% of calls fail normally (timeouts, 5xx, malformed output). During provider incidents, 30%+ for tens of minutes.                    |
| Rate limits?                    | Each provider enforces tokens per minute and requests per minute per API key. We have two providers with comparable models.            |
| Does the user wait on the page? | They may watch results arrive, or close the tab and come back later. Results must persist.                                             |
| Latency expectations?           | First results within ~10s of starting. A 200-paper job should finish in a few minutes.                                                 |
| Cost?                           | LLM spend is the largest infrastructure cost. Avoid paying twice for the same work.                                                    |
| Can users change columns?       | Yes. Editing a column's prompt invalidates that column. Adding a column runs only the new cells.                                       |
| Accuracy?                       | Out of scope for the pipeline itself, but each answer must carry a supporting quote so the user can verify it.                        |

## Reveal mid-interview

Introduce these after the high-level design, one at a time:

1. **Noisy neighbors.** One user just started a 1,000 × 20 job. Everyone else's small jobs stall behind it. What do you do?
2. **Provider outage.** The primary provider's error rate jumps to 40% for 20 minutes. Walk through what happens to in-flight and queued cells, and what the user sees.
3. **Cancellation.** The user realizes a column prompt is wrong and edits it halfway through the job. What happens to in-flight calls you've already paid for?
4. **Deploy mid-job.** Workers are restarted during a deploy. How do you avoid losing or duplicating work?

## Likely probes

- Why a queue instead of calling the LLM from the API server?
- What's your retry policy? How do you avoid a retry storm making an outage worse?
- How do you tell a retryable error from a permanent one (e.g. malformed output vs. a paper too long for the context window)?
- How do you make cell processing idempotent?
- What's cached, and what's the cache key? What if the model version changes?
- How do you get results to the client as they complete? Why that transport?
- How do you enforce provider rate limits across many workers?
- What do you measure? What alerts would you set?
- How does this change at 10x users?

## What a strong answer covers

- **The cell is the unit of work.** A job fans out into independent, idempotent cell tasks with their own state (queued, running, succeeded, failed, cancelled) persisted in a database, not held in memory.
- **Async by default.** The API accepts the job and returns immediately. Workers pull from a durable queue with visibility timeouts or leases, so a crashed worker's tasks get retried.
- **Retries with judgment.** Exponential backoff with jitter, a retry budget per cell, and a distinction between retryable failures (timeouts, 429, 5xx), repairable ones (malformed output: retry with a stricter prompt or validation) and permanent ones (surface to the user).
- **Hedging and timeouts.** Per-call timeouts tied to the expected latency for the paper's length. Possibly hedged requests for tail latency, weighed against cost.
- **Failover and circuit breaking.** Per-provider health tracking, a circuit breaker that shifts traffic to the second provider, and load shedding or deferral when both are degraded.
- **Global rate limiting.** A shared token bucket (e.g. in Redis) per provider key, so workers don't collectively exceed limits.
- **Fairness.** Per-user or per-job concurrency caps, or weighted fair queuing, so large jobs can't starve small ones. Prioritize the first rows a user sees.
- **Caching and dedupe.** Cache results keyed by paper ID + normalized column prompt + model version, so re-runs and identical columns across users don't pay twice.
- **Streaming results to the client.** Cell updates published as they complete (SSE or WebSockets, with a reasoned choice), plus a fetch-by-job endpoint so a returning user gets current state.
- **Cancellation.** Cancelled cells are marked in the DB; workers check before calling and discard results after. Accept that some in-flight spend is lost.
- **Observability.** Per-provider latency and error rates, queue depth and age, cells per job by state, cost per job, and time to first result.

## Tradeoffs worth naming

- Queue per priority vs. one queue with fair scheduling.
- Hedged requests (lower tail latency) vs. cost.
- Chunking long papers (parallel, cheaper per call, may miss cross-section context) vs. one long-context call.
- Caching on normalized prompts (more hits) vs. exact prompts (no false hits).
- Pushing updates via pub/sub vs. clients polling job state.
