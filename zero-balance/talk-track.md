# Zero Balance Problem: 5-minute talk track

Target: about 5 minutes, then hand over to questions. Timings are cumulative. **Show** marks what to put on screen. Everything in [If asked](#if-asked) stays in reserve.

## 0:00 Context and my role (30s)

"I'm going to walk you through the zero balance problem from the early years of Coinbase's GraphQL system. Users would sometimes see a zero balance, and after a high profile incident involving the primary balance on the home screen, we took a more active role in solving it with product teams.

I led the root cause analysis, owned the key design decisions, including how the solution integrated with Relay, and wrote the documentation and training. A member of my team implemented it, and I reviewed it."

## 0:30 Problem (60s)

**Show:** the `?? 0` snippet.

"Here's an example of code that suffers from the problem.

When the balance service failed, Coinbase's nullable GraphQL schema returned a null balance, and client code like this defaulted it to zero. In a crypto app, a user who sees a zero balance may think their funds were stolen, and Coinbase prided itself on being the most trusted name in crypto. So even a momentary zero balance was a serious trust problem.

My investigation found three things.

1. It was widespread: about 100 places in the frontend did this.

2. It was frequent: the balance service had a small but steady error rate, and we estimated it hit thousands of users a day. That's an estimate, because we didn't have client observability for this case.

3. It was often ephemeral: balances are live, so the subscription usually recovered a moment later. This last point ends up shaping the UX."

## 1:30 Constraint and requirements (30s)

"The big constraint in any solution was migration cost. There was a lot of existing code with varied schemas and error states, so anything requiring a migration, especially one involving GraphQL schema and resolvers, would be a tall order.

The UX had three requirements: don't show raw server error messages, keep partial rendering so a single field erroring doesn't break the screen, and handle live data, where a value can be fresh, stale or updating."

## 2:00 Existing solutions (30s)

**Show:** the union-type schema snippet.

"Coinbase had existing options for handling errors. First was the fallback approach we saw earlier. This approach can't tell an error from an empty balance. Alternatively we could model errors in the schema with union types, like in this example, but that means v2 fields and a GraphQL migration which no one wanted to do."

## 2:30 Decision 1: use error metadata (30s)

**Show:** the first `useFragmentWithError` snippet.

"The GraphQL response already tells you which fields errored: each error includes the path of the field it affected. So my first key decision was to build on that metadata. A null with a matching error is an error, and a null without one is organically empty. A new API, `useFragmentWithError`, would give you the data plus a getter for field errors. No schema change, so no backend migration."

## 3:00 Decision 2: normalized error codes (30s)

**Show:** the gRPC to GraphQL error code mapping table.

"Teams also needed to know _why_ a field failed, and many resolvers were hand-mapping gRPC statuses into schema error types. My second decision was to normalize that once, in the GraphQL service. Upstream gRPC statuses map to a small set of standard error codes, like this example, and those codes are included in the error metadata sent to the client. So in most cases, teams no longer needed new schema types to model errors."

## 3:30 Decision 3: staleness (45s)

**Show:** the staleness table.

"The third decision came from the live data requirement. If we showed an error every time one occurred, we'd just swap a flash of a zero balance for a flash of an error. So the UX combines the error metadata with the time since the last good update, and only escalates to an error after some time without one, say 60 seconds. Before that, we show the balance as updating."

## 4:15 How it works (30s)

**Show:** the data flow diagram.

"Next let's talk about how this works on the client. Relay doesn't expose error metadata, so we wired it through ourselves. A network middleware stores each response's errors by query name in a new error metadata store. The hard part is that errors are identified by their path from the root, but a fragment only knows its cache key, not its full path. So I designed the lookup to walk the Relay store along each error path and check whether it ends on a record with a matching cache key. If it does, it returns that error."

## 4:45 Guardrails and outcomes (30s)

**Show:** the preventing regressions table.

"To keep it fixed, we added guardrails. Lint rules block falling back to zero, and new design system balance components have the stale and error logic built in. We coordinated with product teams to fix the violations, and migrated the T0 surfaces to the new components.

Over a quarter, we went from about 100 problem sites to zero, and we didn't see another zero balance incident."

## 5:15 Downsides (30s)

"The main downside of this approach is that it relies heavily on Relay internals. We minimized the risk of an upgrade causing breakage with a large regression test suite.

Relay has since shipped its own field-level error handling, and our team gave input on it. When I left, we were investigating migrating, but there were still differences that blocked it."

_Pause and hand over to questions._

## If asked

| Question                                            | Where to go                                                                                               |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Why is balance nullable? Why not make it required?  | Appendix: Errors and nulls. A required field's null bubbles up and wipes out the whole home query.        |
| What does the error metadata look like?             | Using error metadata. Each error includes the path of the field it affected.                              |
| What does the error code look like on the wire?     | Expanding the API                                                                                         |
| How does the lookup work in code?                   | Appendix: Technical implementation                                                                        |
| What about aliases, arguments, lists, variables?    | Appendix: Edge cases                                                                                      |
| How did you verify it worked without observability? | Outcomes. Incidents and CX tickets were the proxy metrics.                                                |
| Why 60 seconds?                                     | The exact threshold was a product call. What mattered was a grace period for the subscription to recover. |
