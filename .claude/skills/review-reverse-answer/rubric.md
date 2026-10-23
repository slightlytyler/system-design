# Rubric: reverse system design (frontend platform engineer)

Score each dimension from 1 to 4:

- **1 = Missing.** The dimension isn't addressed, or it's addressed in a way that would raise a red flag.
- **2 = Thin.** It's mentioned, but the answer asserts things without reasoning or evidence. The interviewer would have to dig.
- **3 = Solid.** It's clearly addressed with reasoning, and holds up to a first follow-up.
- **4 = Strong.** The answer anticipates the follow-ups and states the reasoning and the costs unprompted. This reads as staff-level.

The weights show how much each dimension usually matters in the interviewer's overall impression. **Overall** is the weighted average, rounded to the nearest 0.5, using judgment. A 1 on a high-weight dimension caps the overall score at 2.5.

---

## 1. Problem framing (weight: high)
Does the answer explain *why* the work was needed?
- It states the concrete pain before the solution: incidents, user impact, developer pain or cost.
- It names who was affected. On a platform team there are two layers: **consumers** (product engineers) and **end users**.
- It gives evidence that the problem was real and significant, such as an incident, data or the number of call sites.
- It defines scope: what was in, what was out, and why.
- **Red flag:** it jumps straight to the solution, or only says "it was a mess".

## 2. Requirements and constraints (weight: medium)
- Functional requirements: what the abstraction had to do.
- Non-functional requirements: performance, bundle size, render cost, correctness, accessibility, SSR/hydration, offline and live data, and security (e.g. not leaking internal error messages).
- Organizational constraints: number of teams, migration cost, backwards compatibility, existing tools it couldn't replace, and who had to agree.
- **Red flag:** constraints show up only when needed to justify a decision after the fact.

## 3. Architecture clarity (weight: high)
- The system can be sketched in about 60 seconds: the components, their boundaries, and how data and control flow between them.
- It shows the **consumer-facing API** (what a product engineer writes) before the internals.
- It's clear where the new pieces sit relative to existing ones, such as the network layer, cache, framework or server.
- **Red flag:** a long wall of implementation before the reader understands the overall shape.

## 4. Tradeoffs and alternatives (weight: highest)
- For each major decision there's at least one named alternative and a reason it was rejected.
- The answer states what the chosen approach **costs**: complexity, coupling to framework internals, performance, maintainability or adoption friction.
- It says what would change the decision ("if X were true, I'd choose Y").
- **Red flag:** "we did X" with no "instead of Y because Z", or alternatives that are straw men.

## 5. Abstraction and API design (weight: high, platform-specific)
- **Ergonomics:** is the common case easy and the correct usage obvious? Does the API make the wrong thing hard?
- **Escape hatches:** what happens when a consumer doesn't fit the abstraction?
- **Leaks:** where does the abstraction leak (framework internals, edge cases), and is that acceptable?
- **Evolution:** versioning, deprecation, codemods, and how the API grew. "Expanding the API" sections should explain *what signal* drove the expansion.
- **Coupling:** reliance on private or undocumented framework internals is a risk. Say how it's mitigated with tests, pinned versions or an upstream path.

## 6. Adoption, migration and guardrails (weight: medium-high, platform-specific)
- How did teams find, adopt and migrate to it: incrementally, with codemods, with lint rules, through docs or office hours?
- Guardrails that keep it correct: lint rules, type-level enforcement, CI checks, ownership.
- Did adoption require convincing people, and how was that done?
- **Red flag:** "we built it and teams used it", with no mechanism described.

## 7. Operability and failure modes (weight: medium)
- Observability: how you know it's working and how you know it broke, e.g. RUM, error tracking, logs or dashboards.
- Failure modes and blast radius, especially for shared infrastructure that every team depends on.
- Rollout safety: feature flags, canaries, kill switches.
- Edge cases are named and their handling is explained. A list of open questions isn't enough.
- Memory, performance and cleanup behavior of any new caches or stores.

## 8. Impact and evidence (weight: high)
- Before-and-after metrics. Rough numbers are fine, but some numbers are needed.
- Adoption numbers, such as call sites, teams or screens.
- User-facing outcomes like fewer incidents, fewer support tickets or better UX, alongside the engineering outcomes.
- **Red flag:** no outcome at all. The story ends with the implementation.

## 9. Ownership and reflection (weight: medium)
- It's clear what *the candidate* did versus the team or other people.
- What they'd do differently, the debt they knowingly took on, and what comes next.
- How the solution relates to where the ecosystem went later (e.g. "Relay has since added X, so today I'd…").

## 10. Communication and structure (weight: medium)
- An upfront outline that the body actually follows.
- Explanation goes from high-level to detailed, and each section earns its place.
- The opening pitch is about 5 minutes, with deeper material marked as "if asked".
- Code snippets are minimal, correct and consistently named. Each one illustrates exactly one point.
- Terminology is consistent and defined on first use.
- **Red flag:** typos or bugs in code, sections that are stubs, or a structure that wanders.

---

## Frontend-platform depth checks
Use these where relevant. They are what a frontend platform interviewer is likely to probe:
- **Data layer:** cache semantics (normalization, keys, invalidation, staleness), live data and subscriptions, optimistic updates, partial data and error propagation.
- **Rendering:** loading, error and empty states; Suspense and error boundaries; avoiding re-renders; SSR and hydration.
- **Schema and contract design:** nullability, error modeling, versioning, the client–server contract and who owns it.
- **Performance:** bundle size, runtime cost of new abstractions (e.g. per-render traversal), and memoization correctness.
- **Developer experience:** type safety, discoverability, lint and codemod support, and documentation.
- **UX:** whether the technical states (error, stale, empty, loading) map to distinct, honest user-facing states.
