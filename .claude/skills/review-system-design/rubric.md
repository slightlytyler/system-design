# Rubric: system design (frontend platform engineer)

Score each dimension from 1 to 4:

- **1 = Missing.** The dimension isn't addressed, or it's addressed in a way that would raise a red flag.
- **2 = Thin.** It's mentioned, but the answer asserts things without reasoning. The interviewer would have to drag it out.
- **3 = Solid.** It's clearly addressed with reasoning, and holds up to a first follow-up.
- **4 = Strong.** The candidate drives the conversation, anticipates the probes, and states the costs of each choice unprompted. This reads as staff-level.

The weights show how much each dimension usually matters in the interviewer's overall impression. **Overall** is the weighted average, rounded to the nearest 0.5, using judgment. A 1 on a high-weight dimension caps the overall score at 2.5.

---

## Phase budget

For a 45-minute interview (scale proportionally for other lengths):

| Phase                                 | Time      |
| ------------------------------------- | --------- |
| Requirements and scope                | 5 min     |
| Estimates and constraints             | 2–3 min   |
| API and data model                    | 5–8 min   |
| High-level architecture               | 8–10 min  |
| Deep dive (1–2 hard problems)         | 10–15 min |
| Scaling, failure modes and wrap-up    | 5 min     |

The deep dive is where seniority shows. An answer that never gets there can't score above 3.

---

## 1. Requirements and scope (weight: high)
- Asks or states clarifying questions before designing: who the users are, which platforms, which features are in scope.
- Separates functional requirements from non-functional ones (latency, availability, consistency, offline, accessibility, internationalization).
- Explicitly cuts scope and says why ("I'll skip notifications and focus on the feed").
- **Red flag:** starts drawing boxes immediately, or treats every feature as in scope.

## 2. Estimates and constraints (weight: medium)
- Rough numbers that matter for the design: users, read/write ratio, payload sizes, update frequency, data growth.
- For frontend-heavy prompts: device and network constraints, item counts on screen, bundle and memory budgets.
- The numbers are actually *used* to justify a decision later.
- **Red flag:** numbers computed and then ignored, or missing where they'd obviously change the design.

## 3. API and data model (weight: high)
- The client–server contract: endpoints or GraphQL schema, request and response shapes, pagination, error shapes.
- The data model: entities, keys, relationships, and where each lives (server DB, cache, client store).
- For frontend-heavy prompts: the client state model (normalized cache vs. per-view state, what's derived, what's persisted).
- **Red flag:** an API that can't support the stated requirements, e.g. no pagination for an infinite feed.

## 4. High-level architecture (weight: high)
- Can be sketched in about 60 seconds: components, boundaries, and data and control flow.
- Walks through one or two core user flows end to end against the diagram.
- Each component earns its place. No boxes added "because systems have them".
- **Red flag:** a diagram of generic boxes (load balancer, cache, DB) with no connection to the requirements.

## 5. Deep dive (weight: high)
- Picks the 1–2 genuinely hard problems in the prompt and goes deep: e.g. feed ranking and pagination, real-time sync and conflict resolution, typeahead latency, virtualization.
- Shows concrete mechanisms, not just names of technologies ("we use Redis" vs. "a sorted set per user keyed by timestamp, trimmed to 500").
- **Red flag:** goes equally shallow on everything, or deep on something easy.

## 6. Tradeoffs and alternatives (weight: highest)
- Each major decision names at least one alternative and why it was rejected: polling vs. SSE vs. WebSockets, SSR vs. CSR, push vs. pull fan-out, normalized vs. denormalized.
- States what the chosen approach costs.
- Says what would change the decision ("if writes were 10x reads, I'd…").
- **Red flag:** "we use X" with no "instead of Y because Z".

## 7. Scale, performance and reliability (weight: medium-high)
- What breaks first as load grows, and how it's addressed.
- Failure modes and degraded states: what the user sees when a dependency is down, retries and backoff, idempotency.
- Caching strategy at each layer (CDN, server, client) with invalidation.
- Observability: what you'd measure to know it's working (e.g. RUM metrics, error rates, p95 latency).

## 8. Communication and driving (weight: medium)
- Gives a roadmap up front and follows it.
- Drives the conversation instead of waiting to be asked; checks in at phase boundaries.
- Fits the phase budget, and leaves time for the deep dive.
- Terminology is consistent and defined on first use.
- **Red flag:** wanders, backtracks repeatedly, or runs out of time before the hard part.

---

## Frontend depth checks
Use these where relevant, weighted by the prompt type. They are what a frontend-focused interviewer is likely to probe:
- **Rendering:** CSR vs. SSR vs. streaming SSR vs. static; hydration cost; code splitting; virtualization for long lists.
- **Data layer:** client cache normalization, keys and invalidation; pagination (cursor vs. offset); optimistic updates and rollback; deduping requests.
- **Real-time:** polling vs. long polling vs. SSE vs. WebSockets; reconnects and missed events; ordering and conflict resolution (OT vs. CRDTs for collaborative editing).
- **Performance:** Core Web Vitals, bundle size, image and asset strategy, prefetching, debouncing and throttling.
- **Offline and network:** service workers, local persistence, request queuing, flaky mobile networks.
- **UX states:** loading, empty, error, stale and partial states are distinct and honest.
- **Accessibility and i18n:** keyboard and screen reader support, focus management, RTL and localization.
- **Security:** XSS and sanitizing user content, CSRF, auth token storage, rate limiting.

## Backend depth checks
For backend-heavy or full-stack prompts:
- **Storage:** SQL vs. NoSQL choice tied to access patterns; indexes; sharding and partition keys.
- **Caching:** what's cached, eviction, invalidation, thundering herd.
- **Async work:** queues, fan-out on write vs. read, idempotent consumers, backpressure.
- **Consistency:** where strong consistency is needed vs. where eventual is fine, and why.
