# Zero Balance Problem

## Intro

The zero balance problem is a classic Coinbase problem from the early years of the GraphQL system. The problem had existed in many forms but started popping up more as we progressed the migration to GraphQL. After a high profile incident involving the primary balances on the homepage we took a more active role in working with product teams to solve it.

## Problem

Our initial investigation discovered that the problem occurred when the balance service failed, Coinbase's nullable GraphQL schema returned null balances, and client code across the app defaulted these nulls to zero via nullish coalescing like `balance ?? 0`, so users saw a $0 balance.

```tsx
const data = useFragment(graphql`...`, ref);
const balance = data.balance ?? 0;
//
<PrimaryBalance balance={balance} />;
```

There were hundreds of places across frontend with this sort of behavior. When we looked at error rates for the balance service we found a small but steady error rate. We didn't have observability into this exact client behavior but based on things like client RPS, service error rate, etc, we assumed this was happening to thousands of users every day.

It's also important to note that often this zero balance case was ephemeral where the user would see the zero balance just for an instant before it was updated and working again. Balances were updated live via GraphQL subscriptions so the initial query would fail but eventually the subscription would kick in and succeed.

## Scope

To address this problem we need to figure out:

1. The expected UX in these cases
2. What are the technical blockers to that UX
3. How to prevent new problems from occurring once we have a solution

#### Migration cost

We need to be mindful that there is a lot of code already in place with a decent amount of variation between the GraphQL schema being used, the various error states, etc. Any solution involving a migration will be a tall order.

## Expected UX

#### Careful error messaging

Rendering a server created or unlocalized error message to the end user does not help. We need to be careful when and how we show error messages.

#### Partial rendering

In GraphQL architecture we expect that the application should be partially renderable even if some of the data is missing. This creates problems for critical sections of the UI like the primary balance. What do we render when balance is totally missing or errored?

#### Live data

Balances are hooked up to live data so it's more complex than just having a few terminal states. We need UX that expresses to the user when a value is up to date vs stale and when it's being updated.

## Finding an alternative

#### Existing error handling

At CB we had some existing query error handling best practices that boiled down to:

    a. Check if a field is present and either render a fallback or nothing at all.
    b. If you have more complex states (like an organic null state, or explicit error states) use custom schema to solve it.

Here's what b. looks like in practice:

```graphql
type Query {
  primaryBalanceV2: BalanceResult
}

union BalanceResult = Balance | ServiceError | GenericError
```

Using union types we can model explicit error states that the client can handle individually. The problem is this is a breaking change and would require migration (notice the `v2` field name).

Any schema migration comes at significant cost in terms of staffing hours, trust, morale, etc. Many migrations that start are never completed. No one is excited about having to make a v2 field or model errors this way unless the data model is truly complex.

#### Using error metadata

Ok so we can see that using a simple fallback doesn't work, and creating new schema types/fields and migrating to them would be an enormous problem.

If we take a look at an example of a GraphQL response with errors we can see there's additional metadata available in the response:

```js
{
    data: {
        primaryBalance: {
            balance: null
        },
        topAssets: [...]
    },
    errors: [{ path: ['primaryBalance', 'balance'], message: '...' }]
}
```

If we can read this error metadata we could enable new UX. At least differentiating between organically null (no associated error) and null due to error. Here's what an API could look like:

```tsx
const [getBalanceError, data] = useFragmentWithError(graphql`...`, ref);

const balanceError = getBalanceError("balance");

if (data.balance == null && balanceError != null) {
  return <BalanceError />;
}
if (data.balance == null) {
  return <DepositFundsCTA />;
}
return <PrimaryBalance balance={data.balance} />;
```

#### Expanding the API

During our investigation of the existing patterns we noticed a lot of repeated schema for handling error states. While some schema had custom error states explicitly as part of their model much of the schema error handling was dedicated to handling the common cases like service error, connectivity issues, authorization, etc. Many resolvers had code that was directly mapping various gRPC error codes into GraphQL schema error types.

We realized that we could use this new abstraction to solve for this problem as well. This would allow devs to access the error code associated with their fragment/field and render an appropriate localized experience. All without having to create v2 resolvers or migrating schema.

```tsx
const [getBalanceError, data] = useFragmentWithError(graphql`...`, ref);

const balanceError = getBalanceError("balance");

if (data.balance == null && balanceError != null) {
  switch (balanceError.code) {
    case "SERVER_ERROR":
      // ...
      break;
    case "AUTHENTICATION_ERROR":
      // ...
      break;
  }
}
```

#### Live data and staleness

Error metadata tells us _why_ a balance is missing, but on its own it would have made the UX worse. Remember that most zero balances were ephemeral: the initial query failed, but the subscription recovered a moment later. If we showed an error every time we saw error metadata, we'd just replace a flash of $0 with a flash of an error message.

So the balance UX combines two signals: the error metadata, and how long it's been since we last received a good balance update.

| Balance       | Error metadata | Last good update        | UX                                     |
| ------------- | -------------- | ----------------------- | -------------------------------------- |
| present       | –              | recent                  | Balance                                |
| present       | –              | over `<N>s` ago (stale) | Balance faded out with a shimmer       |
| missing       | yes            | under 60s               | Shimmer placeholder, expect a recovery |
| missing       | yes            | none in 60s             | Error state, message chosen by code    |
| missing       | none           | –                       | Deposit funds CTA (organically empty)  |

The key decision was to only escalate to an error after 60s without a balance update. Before that, we assume the subscription will fix it, and we show the user that the value is updating instead of telling them something is wrong. When an update arrives, the balance is fresh again and the error no longer matters.

This logic is subtle enough that we didn't want hundreds of call sites reimplementing it, which is part of why it ended up in the design system's balance components (below).

## Technical blockers

#### Blocked by Relay

Unfortunately Relay doesn't provide access to the error metadata. In theory this makes sense since the error metadata is for debugging not rendering. In practice this left a hole in Relay's error handling (which has started to be addressed in more recent versions).

To access this metadata we would need to wire it up from the source to the UI.

#### Extracting error metadata from responses

The first part of the problem is extracting error metadata from the response and indexing it. This part is straightforward. First we create a new ErrorMetadataStore and attach it to our customized GraphqlEnvironment class.

#### Reading error metadata via fragments

The real challenge of this problem is that the information we have available in the network response (query name, field paths) does not directly map to the metadata available when reading fragment data out of the store (fragment name, fragment id, cache key).

The question is when useFragment is called how do we know which error is associated with it? A fragment or field has a cache key associated with it but we don't know its full path from the query root.

To figure out what key is associated with which error path we have to start from the other direction. Basically what we want is path directed traversal that translates error paths into relay keys. Once we have a match we return the associated error.

#### New data flow

In the new data flow the upstream gRPC status is mapped to a normalized error code in the graphql service, where it's included in the error metadata, which is then extracted in the client in a middleware, and set in a new error metadata store. The client reads from this store to determine which UX to present.

![Alt text](./images/new-data-flow.png)

#### Addressing additional concerns

The new APIs along with the documentation and training we created gave developers all the tools to avoid this problem but it didn't address the systemic problems already in the codebase. To address this we introduced new linting rules that blocked the nullish coalescing to zero.

In addition we worked with the design system team to create balance components that encapsulated the logic around showing balances, stale states, and error states, so that individual product teams wouldn't have to reimplement them.

## Outcomes

The deliverables were:

1. New hooks API for reading error metadata
2. New linting rules that disallow nullish coercion to zero.
3. New design system components for displaying stale balances and errors.
4. Documentation and training materials

We were able to systematically eliminate the zero balance by:

1. Removing nullish coalescing violations as detected by the lint rule.
2. Migrating surfaces to the new design system components

This was verified by the reduction in incidents and CX tickets. Unfortunately there wasn't a straightforward way to detect this problem via client observability so we used proxy metrics.

We were also able to simplify schema design for new use cases due to the new APIs covering most cases.

## Appendix

### HomeScreenQuery

```graphql
query HomeScreenQuery {
  primaryBalance {
    currency
    balance
  }
  topAssets(first: 3) {
    asset {
      ticker
      price
    }
  }
  # more below the fold content
}
```

## GraphQL Architecture

In CB's GraphQL architecture the GraphQL service sits in between the client and upstream services. The client loads all the data for a screen via a single query where the GraphQL service handles making the requests to the services and stitching the responses together.

![Alt text](./images/graphql-v1-arch.png)

On the client Relay provides an in memory normalized cache. The cache is essentially a mapping between records and their Relay cache key. The keys look something like this:

```ts
// when a record has an explicit `id` field
"Account:12345";
// when a record does not have an id the key is built from its parent key, field name, and arguments
"client:root:primaryBalance";
"client:root:topAssets(first:3)";
```

## Errors and nulls

What happens to the HomeScreenQuery when the balance service is down? In GraphQL a response with errors is still a 200 where we resolve as much data as possible. Depending on how we structure the schema we get very different responses.

Here's an example where we use required types which avoids nullable types:

```graphql
type Query {
  primaryBalance: Balance!
  topAssets(first: Int): [Asset!]!
}

type Balance {
  currency: String!
  balance: String!
}
```

The problem is balance service going down means we can't resolve `primaryBalance.balance` which results in a response like:

```js
{
    data: null,
    errors: [{ path: ['primaryBalance', 'balance'], message: '...' }]
}
```

The `balance` and `primaryBalance` fields are required so the null bubbles to the top level which results in the `topAssets` field getting clobbered.

Ok let's consider a different schema:

```graphql
type Query {
  primaryBalance: Balance
  topAssets(first: Int): [Asset]
}

type Balance {
  currency: String
  balance: String
}
```

This time when balance service goes down we get a response like:

```js
{
    data: {
        primaryBalance: {
            balance: null
        },
        topAssets: [...]
    },
    errors: [{ path: ['primaryBalance', 'balance'], message: '...' }]
}
```

We called this concept "nullable schema" and it ends up being an axiom of GraphQL system design. This pattern allows you to update as much of the UI as possible and prevents individual services from breaking the response.

## Technical implementation

The first part of the problem is extracting error metadata from the response and indexing it. This part is straightforward. First we create a new ErrorMetadataStore and attach it to our customized GraphqlEnvironment class:

```ts
class ErrorMetadataStore {
  // Map of query name => response errors
  cache = new Map<string, GraphqlResponseError[]>();

  extractErrorsFromResponse(json) {
    // operationName is a custom extension added by our GraphQL service
    const operationName = json.extensions.operationName;
    const errors = json.errors;
    if (errors) {
      this.cache.set(operationName, errors);
    } else {
      // cleanup
      this.cache.delete(operationName);
    }
  }
}

class GraphqlEnvironment {
  errorMetadataStore = new ErrorMetadataStore();
}
```

Next we integrate the error metadata store into the network layer via a middleware:

```ts
const extractErrors = (graphqlEnv) => (next) => async (url, init) => {
  const response = await next(url, init);
  // Clone so the body can still be read by Relay
  const json = await response.clone().json();
  graphqlEnv.errorMetadataStore.extractErrorsFromResponse(json);
  return response;
};
```

The real challenge of this problem is that the information we have available in the network response (query name, field paths) does not directly map to the metadata available when reading fragment data out of the store (fragment name, fragment id, cache key).

Relay has two strategies for creating cache keys:

```
// Cache key structure
<recordId> // 47ac10b-58cc-4372-a567-0e02b2c3d479
<parentRecordId>:<fieldKey>:<indexKey> // 47ac10b-58cc-4372-a567-0e02b2c3d479:foo:bar(first:10):2:baz
```

The question is when useFragment is called how do we know which error is associated with it? It's possible to figure out this mapping but it requires walking the Relay store and finding which key is associated with which path.

Basically what we want is path directed traversal that translates error paths into relay keys. With that map in hand we can lookup error metadata from the useFragment calls.

```ts
class ErrorMetadataStore {
  constructor(relayStore) {
    this.relayStore = relayStore;
  }

  cache = new Map<string, GraphqlResponseError[]>();

  extractErrorsFromResponse(json) { ... }

  findErrorMetadata(queryName, fragmentId, fieldPath) {
    const fieldSegments = fieldPath.split(".");
    const errors = this.cache.get(queryName) ?? [];
    for (const err of errors) {
      // The error path ends with the errored field, everything before it
      // leads to the record that owns that field
      const erroredField = err.path.slice(-fieldSegments.length).join(".");
      if (erroredField !== fieldPath) {
        continue;
      }
      // Walk the Relay store from the root along the error path to find
      // the owning record's cache key
      let recordId = "client:root";
      for (const segment of err.path.slice(0, -fieldSegments.length)) {
        recordId = getLinkedRecordId(this.relayStore, recordId, segment);
        if (recordId == null) {
          break;
        }
      }
      if (recordId === fragmentId) {
        return err;
      }
    }
    return null;
  }
}
```

```tsx
function useFragmentWithError(fragment, ref) {
  const data = useFragment(fragment, ref);
  const env = useGraphqlEnvironment();
  // Relay internals: the owning query's name and the fragment's record id
  const queryName = ref.__fragmentOwner.node.params.name;
  const fragmentId = ref.__id;
  // Reset lookups whenever the fragment data changes so errors don't
  // outlive a refetch or subscription update
  const errorIndex = useMemo(
    () => new Map<string, GraphqlResponseError | null>(),
    [data],
  );
  const getError = (path: string) => {
    if (!errorIndex.has(path)) {
      errorIndex.set(
        path,
        env.errorMetadataStore.findErrorMetadata(queryName, fragmentId, path),
      );
    }
    return errorIndex.get(path);
  };
  return [getError, data] as const;
}
```

## Edge cases

There's a lot of edge cases not captured here. What happens when a field has arguments? What happens when a field is aliased? What happens when a field is a list type? What happens when the same query is in flight with different variables, since errors are keyed by query name? The `findErrorMetadata` function gets complicated fast and requires significant investigation and a suite of unit tests to validate it.
