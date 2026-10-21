# Zero Balance Problem

## Intro

The zero balance problem is a classic Coinbase problem from the early years of the GraphQL system. The problem had existed in many forms but started popping up more as we progressed the migration to GraphQL. After a high profile incident involving the primary balances on the homepage we took a more active role in working with product teams to solve it.

## Outline

Today I'm going to tell you about

1. The problem and our initial investigation into it.
2. How the GraphQL architecture is related to this problem and the deeper root cause.
3. The existing solutions for GraphQL error handling and an alternative we designed.
4. Technical implementation of this new solution
5. Followups, pushing the solution further, and guardrails be built.

## Problem

In many places in the application we had code that looked something like this:

```tsx
const balance = ...;
//
<PrimaryBalance balance={balance ?? 0} />
```

PrimaryBalance.balance is required but the balance value is nullable so a fallback is provided.

In this first example its pretty obvious that we're falling back to zero. but it might not always be so obvious:

```tsx
const data = useFragment(graphql`...`, ref) ?? { balance: 0 };
//
<PrimaryBalance balance={data.balance} />;
```

In the second example we're calling useFragment from Relay which reads data from the cache. Once again we're falling back to zero

Or:

```tsx
function usePrimaryBalance(ref) {
  const data = useFragment(graphql`...`, ref);
  return { balance: data.balance ?? 0 };
}
//
const { balance } = usePrimaryBalance(ref);
//
<PrimaryBalance balance={balance} />;
```

In this third example we're calling useFragment in a hook and providing the fallback in that hook as well. The caller of this hook likely doesn't know about this behavior.

A simple answer would be to say that the balance value is required and should always be available but thats not possible without severly degrading the UX of the homepage

## GraphQL Architecture

In CB's GraphQL architecture the GraphQL service sits in between the client and upstream services. The client loads all the data for a screen via a single query where the GraphQL service handles making the requests to the servivces and stiching the responses together.

The client has an in memory cache which powers the UI.

## HomeScreenQuery example

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

## Error handling

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

## Finding an alternative

Earlier I showed an example of an error response that included some additional metadata in the `errors` array. What could we use this metadata for?

A kneejerk reaction would be to show the error message to the user in lieue of the balance or beside it. This is a bad idea for a number of reasons but top of the list is the message isn't obfuscated or otherwise safe to show to users and showing it to users will likely just confuse them more.

However the presence of the error metadata let's us differentiate between organically null (no associated error metadata) and null due to error. This would allow for a new UX:

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

## Technical implementation

Unfortunately Relay doesn't provide access the error metadata. In theory this makes sense since the error metadata is for debugging not rendering. In practice this left a hole in Relay's error handling (which has started to be addressed in more recent version).

To access this metadata we had to wire it up from the source to the UI. Relay has a cache but does not provide general write access to it, so we'll have to hold the metadata in memory somewhere.

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

## Expanding the API

As we began integrating this new API in the client one thing we found is that the presence of error metadata wasn't always sufficient info. In some cases we needed additional info on the cause of the error. Was it a service error? Was it an authentication error? Etc. There was a set of base line error cases that we saw repeated throughout the codebase and we captured these in a new error code API

```tsx
const [getBalanceError, data] = useFragmentWithError(graphql`...`, ref)

const balanceError = getBalanceError('balance');

if (balanceError) {
    switch (balance.code) {
        case 'SERVER_ERROR':
            // ...
        case 'AUTHENTICATION_ERROR':
            // ...
    }
}

return <PrimaryBalance balance={data.balance}>
```

This new error code API provided a directive alternative to the custom schema approach to modelling errors. On the server we built the error mapping into the network layer to automatically apply error codes based on the upstream GRPC reponse and also provided methods for manually setting error codes for edge cases where there wasn't a direct mapping between the upstream response and GraphQL error code.

## Live data

One thing not mentioned in this discussion is live data. In these examples we treat receiving an error from the query as a terminal state. This is not a correct way to model this because balances are live so are constantly reveiving updates from an associated GraphQL subscription.

As we got deeper into this problem we discovered there was confusion between what's an error state vs some sort of state that expresses that the balance is stale but is attempting to be updated. We don't want to show the user an error if it's immediately going to be fixed by an update. We only want to show errors for "sufficiently stale" data.

What we really needed in this case is a UX that communicates this nuance i.e. if a balance is present but stale show it's value as faded out with a shimmer. Only show errors if we haven't received a balance update in 60s.

## Edge cases

There's a lot of edge cases not captured here. What happen when a field has arguments? What happens when a field is aliased? What happens when a field is a list type? The `findErrorMetadata` function gets complicated fast and requires
significant investigation and a suite of units tests to validate it.

## Guardrails

Observability, lint rules, etc
