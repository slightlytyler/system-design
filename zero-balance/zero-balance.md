# Zero Balance Problem

## Intro

The zero balance problem is a classic Coinbase problem from the early years of the GraphQL system. The problem had existed in many forms but started popping up more as we progressed the migration to GraphQL. After a high profile incident involving the primary balances on the homepage we took a more active role in working with product teams to solve it.

## Problem

Our initial investigation discovered that the problem occurred when the balance service failed, Coinbase's nullable GraphQL schema returned null balances, and client code across the app defaulted these nulls to zero via nullish coalescing like `balance ?? 0`, so users saw a $0 balance.

```tsx
const data = useFragment(graphql`...`, ref);
const balance = data.balance ?? 0;
//
<PrimaryBalance balance={data.balance} />;
```

There were hundreds of places accross frontend with this sort of behavior. When we looked at error rates for the balance service we found a small but steady error rate. We didn't have observability into this exact client behavior but based on things like client RPS, service error rate, etc, we assumed this was happening to thousands of users every day.

It's also important to note that often this zero balance case was ephemeral where the user would see the zero balance just for an instant before it was updated and working again. Balances were updated live via GraphQL subscriptions so the initial query would fail but eventually the subscription would kick in and succeed.

## Scope

To address this problem we need figure out:

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
  primaryBalanceV2: BalanceResults
}

union BalanceResult = Balance | ServiceError | GenericError
```

Using union types we can model explicit error states that the client handle individually. The problem is this is a breaking change and would require migration (notice the `v2` field name).

Any schema migration comes at significant cost in terms of staffing hours, trust, morale, etc. Many migrations that start are never completeled. No one is excited about having to make a v2 field or model errors this way unless the data model is truly complex.

#### Finding an alternative

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
    errors: [{ path: 'primaryBalance.balance', message: '...' }]
}
```

If we can read this error metadata we could enable new UX. At least differentiating between organically null (no associated error) and null due to error. Here's what an API could look like:

```tsx
const [getBalanceError, data] = useFragmentWithError(graphql`...`, ref)

const balanceError = getBalanceError('balance');

if (data.balance == null and balanceError !== null) {
    return <BalanceError />
}
if (data.balance == null) {
    return <DepositFundsCTA />
}
return <PrimaryBalance balance={data.balance}>
```

#### Expanding the API

During our investigation of the existing patterns we noticed a lot of repeated schema for handling error states. While some schema had custom error states explicitly as part of their model much of the schema error handling was dedicated to handling the common cases like service error, connectivity issues, authorization, etc. Many resolvers had code that was directly mapping various gRPC error codes into GraphQL schema error types.

We realized that we could use this new abstraction to solve for this problem as well. This would allow devs to access the error code associated with their fragment/field and render an appropriate localized experience. All without having to create v2 resolvers or migrating schema.

```tsx
const [getBalanceError, data] = useFragmentWithError(graphql`...`, ref);

const balanceError = getBalanceError("balance");

if (data.balance == null && balanceError) {
  switch (balance.code) {
    case "SERVER_ERROR":
    // ...
    case "AUTHENTICATION_ERROR":
    // ...
  }
}
```

## Technical blockers

#### Blocked by Relay

Unfortunately Relay doesn't provide access the error metadata. In theory this makes sense since the error metadata is for debugging not rendering. In practice this left a hole in Relay's error handling (which has started to be addressed in more recent version).

To access this metadata we would need to wire it up from the source to the UI.

#### Extracting error metadata from responses

The first part of the problem is extracting error metadata from the response and indexing it. This part is straightforward. First we create a new ErrorMetadataStore and attach it to our customized GraphqlEnvironment class.

#### Reading error metadata via fragments

The real challenge of this problem is that the information we have available in the network response (query name, field paths) does not directly map to the metadata available when reading fragment data out of the store (fragment name, fragment id, cache key).

The question is when useFragment is called how do we know which error is associated with it? A fragment or field has a cache key associated with it but we don't know it's full path from the query root.

To figure out what key is associate with which error path we have to start from the other direction. Basically what we want is path directed traversal that translates error paths into relay keys. Once we have a match we return the associated error.

#### New data flow

In the new data flow the upstream gRPC status is mapped to a normalized error code in the graphql service, where it's included in the error metadata, which is then extracted in the client in a middleware, and set in a new error metadata store. The client reads from this store to determine which UX to present.

![Alt text](./images/new-data-flow.png)

## Edge cases

There's a lot of edge cases not captured here. What happen when a field has arguments? What happens when a field is aliased? What happens when a field is a list type? The `findErrorMetadata` function gets complicated fast and requires
significant investigation and a suite of units tests to validate it.

## Guardrails

Observability, lint rules, etc

## Wrapping up

So to wrap up we worked with product to understand the problem, designed alternatives, implemented them, and then worked with those times to integrate the solutions, train them on them, and provide ongoing support.

The deliverables were:

1. New hooks API for reading error metadata
2. New design system components for displaying stale balances and errors.
3. Documentation and training materials

## Appendix

### HomeScreenQuery

```graphql
query HomeScreenQuery {
  primaryBalance {
    currency
    value
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

In CB's GraphQL architecture the GraphQL service sits in between the client and upstream services. The client loads all the data for a screen via a single query where the GraphQL service handles making the requests to the servivces and stiching the responses together.

![Alt text](./images/graphql-v1-arch.png)

On the client Relay provides an in memory normalized cache. The cache is essentially a mapping between records and their Relay cache key. The keys look something like this:

```ts
// when a record has an explicit `id` field
"Account:12345";
// when a record does not have an id the key is built using path/index/arguments.
"client:root:primaryBalance.balance";
```

## Errors and nulls

What happens to the HomeScreenQuery when the balance service is down? In GraphQL a response with errors is still a 200 where we resolve as much data as possible. Depending on how we structure the schema we get very different responses.

Here's an example where we use required types which avoids nullable types:

```graphql
type Query {
  primaryBalance: Balance!
  topAssets(first: Number): [Asset!]!
}
```

The problem is balance service going down means we can't resolve `primaryBalance.balance` which results in a response like:

```js
{
    data: null,
    errors: [{ path: 'primaryBalance.balance', message: '...' }]
}
```

The `primaryBalance` field is required so the null bubbles to the top level which results in the `topAssets` field getting clobbered.

Ok let's consider a different schema:

```graphql
type Query {
  primaryBalance: Balance
  topAssets(first: Number): [Asset]
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
    errors: [{ path: 'primaryBalance.balance', message: '...' }]
}
```

We called this concept "nullable schema" and it ends up being an axiom of GraphQL system design. This pattern allows you to update as much of the UI as possible and prevents individual services from breaking the response.

## Technical implementation

The first part of the problem is extracting error metadata from the response and indexing it. This part is straightforward. First we create a new ErrorMetadataStore and attach it to our customized GraphqlEnvironment class:

```ts
class ErrorMetadataStore {
  cache = new Map<string, { errors: GraphqlResponseError[] }>();

  extractErrorsFromResponse(response) {
    const operationName = response.extensions.operationName;
    const errors = response.errors;
    if (errors) {
      this.cache.set(operationName, errors);
    } else {
      // cleanup
      this.cache.delete(operationName);
    }
  }
}

class GraphqlEnvironment {
  // Map of query name => response errors
  errorMetadataStore = new ErrorMetadataStore();
}
```

Next we integrate the error metadata store into the network layer via a middleware:

```ts
const extractErrors = (graphqlEnv) => (next) => async (url, init) => {
  const response = await next(url, init);
  graphqlEnv.errorMetadataStore.extractErrorsFromResponse(response);
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

    cache = new Map<string, { errors: GraphqlResponseError[] }>

    extractErrorsFromResponse(response) { ... }

    findErrorMetadta(queryName, fragmentId, path) {
        const selectedCacheKey = buildCacheKey(fragmentId, path);
        const errors = cache.get(queryName);
        // Iterate over errors
        for (const err in errors) {
            const path = err.path;
            let obj = this.relayStore.getRecord('client:root');
            // Iterate over path
            for (let i = 0; i < path.length; i++) {
                const p = path[i];
                obj = this.relayStore.getValue(...);
            }
            const cacheKey = obj.__id;
            if (cacheKey === selectedCacheKey) {
                return err;
            }
        }
    }
}
```

```tsx
function useFragmentWithErrors(fragment, ref) {
  const data = useFragment(framgnet, ref);
  const env = useGraphqlEnvironment();
  const errorIndex = useRef<Map<string, GraphqlResponseError>>(new Map());
  const queryName = ref.__fragmentOwner.node.params.name;
  const fragmentId = fragmentRef.__id;
  const getError = (path) => {
    const indexedError = errorIndex.get(path ?? "");
    if (indexedError) {
      return indexedError;
    }
    const foundError = env.errorMetadataStore.findErrorMetadata(
      queryName,
      ref,
      path,
    );
    errorIndex.set(path ?? "", foundError);
    return foundErrror;
  };
  return [getError, data];
}
```
