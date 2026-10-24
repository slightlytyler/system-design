# Interviewer notes: AI results table

Spoilers. Read these only after attempting the problem, or have someone play the interviewer with them.

## Answers to clarifying questions

Give these only when asked.

| Question                          | Answer                                                                                                                         |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Platforms?                        | Desktop web only. React.                                                                                                       |
| How big is a table?               | Typically 50–200 rows × 3–10 columns. The largest are 1,000 rows × 20 columns (20k cells).                                     |
| How big is a cell?                | Answers are usually a short phrase or a number, sometimes a paragraph. Each has one or more supporting quotes with locations.  |
| Does the backend stream?          | The backend can push cell-level events (queued, started, token deltas, completed, failed). You can propose the API shape.     |
| Token streaming per cell?         | Nice to have in the table. Required in the summary panel and chat.                                                             |
| Can users edit cells?             | Yes, they can override an AI answer. Overrides must survive regeneration unless the user explicitly discards them.            |
| Collaboration?                    | Tables can be shared read-only. Real-time multi-user editing is out of scope.                                                  |
| Sorting and filtering?            | Users sort and filter on AI values, e.g. "sample size > 100". Rows whose cells aren't done yet still need a sensible place.    |
| Persistence?                      | Users close the tab and come back later. The table must restore from the server, including in-progress cells.                |
| Export?                           | CSV export of the current view.                                                                                                |

## Reveal mid-interview

Introduce these after the high-level design, one at a time:

1. **Edit a column mid-run.** The user edits a column's prompt while 300 of its cells are in flight. What happens to those cells, the ones already done, and any manual overrides?
2. **Update storm.** A 1,000 × 20 table is running and hundreds of cell events arrive per second. The table starts to jank. Where is the time going, and how do you fix it?
3. **Reconnect.** The user's laptop sleeps for 10 minutes mid-run. What does the UI show when it wakes up, and how does it catch up?
4. **Second feature.** The chat team wants to reuse your work. What do they get, and what stays specific to the table?

## Likely probes

- What does the state for a single cell look like? Draw its state machine.
- Where does server state live vs. UI state? What library or pattern, and why?
- How do you avoid re-rendering the whole table when one cell updates?
- How do you sort a column where half the values aren't generated yet?
- What happens if events arrive out of order or are duplicated?
- What does a failed cell look like? Can the user retry one cell, a column, or all failures?
- How do you show the user that an answer is trustworthy (quotes, sources)?
- How do you keep AI-specific logic from leaking into every component?
- How would you test this?

## What a strong answer covers

- **A normalized client store.** Entities for papers, columns and cells keyed by `(paperId, columnId)`, with each cell tracking its column version. Views subscribe to slices, not the whole table.
- **An explicit cell state machine.** States like idle, queued, streaming, done, failed, stale and overridden, with defined transitions. Events from the server drive transitions; components render from state, not from ad hoc flags.
- **Versioned columns.** Editing a column bumps its version. Cells from older versions are marked stale or discarded; in-flight results for an old version are ignored on arrival. Overrides are kept and flagged.
- **A resumable event stream.** SSE or WebSockets (with a reasoned choice) carrying idempotent events with sequence numbers or cell versions. On reconnect, refetch a snapshot or resume from the last sequence number.
- **Render performance.** Batch incoming events per animation frame, row and column virtualization, memoized cells subscribed by key, and throttled token streaming in the table.
- **Sorting and filtering with partial data.** Define where pending cells sort (e.g. last, grouped), and avoid rows jumping around as values arrive, e.g. by freezing the order until the user re-sorts.
- **A reusable AI content primitive.** A shared layer (hook or store module plus components) for "a piece of AI-generated content": its lifecycle state, streaming, error and retry, citations, and staleness. The table, summary panel and chat compose it instead of reimplementing it. Table-specific concerns (layout, sorting, columns) stay in the table.
- **Honest UX states.** Distinct visuals for queued, generating, failed, stale and overridden, with retry at cell, column and job level, and a supporting quote one click away.
- **Testing.** Unit tests for the state machine and reducers, fixture-driven event replays for ordering and reconnects, and performance budgets for the large-table case.

## Tradeoffs worth naming

- A normalized global store vs. server-state caching libraries (e.g. TanStack Query, Relay) vs. per-component state.
- SSE vs. WebSockets vs. polling for cell events.
- Token streaming in every cell (feels alive) vs. completion-only updates (cheaper to render).
- Live re-sorting vs. stable order while results arrive.
- A generic AI content abstraction vs. letting each feature own its own logic until patterns settle.
