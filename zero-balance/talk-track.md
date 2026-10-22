# Zero Balance Problem: 5-minute talk track

> **Generated file. Don't edit this document directly.** It's derived from [zero-balance.md](./zero-balance.md). Make changes there, then regenerate this talk track.

Target: about 5 minutes, then hand over to questions. Timings are cumulative. **Show** marks what to put on screen. Everything in [If asked](#if-asked) stays in reserve.

## 0:00 Context and my role (30s)

"This is the zero balance problem at Coinbase, from the early years of our GraphQL system. Users would sometimes see a $0 balance, and after a high profile incident involving the primary balance on the homepage, we took a more active role in solving it with product teams.

I led the root cause analysis, owned the key design decisions, including how the solution integrated with Relay, and wrote the documentation and training. A member of my team implemented it, and I reviewed it."

## 0:30 Problem (60s)

**Show:** the `?? 0` snippet.

"When the balance service failed, our nullable schema returned a null balance, and client code like this defaulted it to zero.

My investigation found three things. It was widespread: about 100 places in the frontend did this. It was frequent: the balance service had a small but steady error rate, and we estimated it hit thousands of users a day. That's an estimate, because we had no client observability for it. And it was often ephemeral: balances are live, so the subscription usually recovered a moment later. That last point ends up shaping the UX."

## 1:30 Constraint and requirements (30s)

"The big constraint was migration cost. There was a lot of existing code with varied schemas and error states, so anything requiring a migration would be a tall order.

The UX had three requirements: don't show raw server error messages, keep partial rendering so one failed field doesn't break the screen, and handle live data, where a value can be fresh, stale or updating."

## 2:00 Decision 1: use error metadata (60s)

**Show:** the response with `errors`, then the first `useFragmentWithError` snippet.

"Our existing options were a fallback, which can't tell an error from an empty balance (that's how `?? 0` happened), or modeling errors in the schema with union types, which means v2 fields and a migration.

But the GraphQL response already tells you which fields errored, through the error path. So my first key decision was to build on that metadata. A null with a matching error is an error, and a null without one is organically empty. `useFragmentWithError` gives you the data plus a getter for field errors. No schema change, so no migration."

## 3:00 Decisions 2 and 3: error codes and staleness (60s)

**Show:** the staleness table.

"Teams also needed to know _why_ a field failed, and many resolvers were hand-mapping gRPC errors into schema types. My second decision was to normalize that once, in the GraphQL service, so every error carries a code in its extensions.

The third decision came from the live data. If we showed an error every time, we'd just swap a flash of $0 for a flash of an error. So the UX combines the error metadata with the time since the last good update, and only escalates to an error after 60 seconds without one. Before that, we show the balance as updating."

## 4:00 How it works (30s)

**Show:** the data flow diagram.

"Relay doesn't expose error metadata, so we wired it through ourselves. A network middleware stores each response's errors by query. The hard part is that errors are identified by their path from the root, but a fragment only knows its cache key. So I designed the lookup to walk the Relay store along each error path and check whether it lands on the fragment's record."

## 4:30 Guardrails and outcomes (30s)

"To keep it fixed, we added lint rules that block falling back to zero, and built design system balance components with the stale and error logic built in. We coordinated with product teams to fix the violations, and migrated the T0 surfaces to the new components.

Over a quarter, we went from about 100 problem sites to zero, and we didn't see another zero balance incident."

## 5:00 Downsides (30s)

"The main downside is that it relies on Relay internals, which we contained with a large regression test suite for Relay upgrades. The API also got wide, with a `WithError` variant for each way of reading data.

Relay has since shipped its own field-level error handling, and our team gave input on it, including semantic non-null. When I left, we were investigating migrating, but there were still differences that blocked it."

_Pause and hand over to questions._

## If asked

| Question                                             | Where to go                                                                                           |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Why is balance nullable? Why not make it required?   | Appendix: Errors and nulls. A required field's null bubbles up and wipes out the whole home query.    |
| What does the union-type alternative look like?      | Finding an alternative: Existing error handling                                                       |
| What does the error code look like on the wire?      | Expanding the API                                                                                     |
| How does the lookup work in code?                    | Appendix: Technical implementation                                                                    |
| What about aliases, arguments, lists, variables?     | Appendix: Edge cases                                                                                  |
| How did you verify it worked without observability?  | Outcomes. Incidents and CX tickets were the proxy metrics.                                            |
| Why 60 seconds?                                      | The exact threshold was a product call. What mattered was a grace period for the subscription to recover. |
