# HTTP Streams vs. WebSockets: 5-minute talk track

Target: about 5 minutes, then hand over to questions. Timings are cumulative. **Show** marks what to put on screen. Everything in [If asked](#if-asked) stays in reserve.

## 0:00 Context and my role (45s)

"I'm going to walk you through the decision to standardize GraphQL live data at Coinbase on HTTP streams rather than WebSockets.

Some context: before around 2018, Coinbase supported three assets, BTC, ETH, and LTC. Coinbase's 'Amazon of crypto' strategy grew that to thousands of assets and far more users, but by 2021 the live data infrastructure hadn't caught up. It was built for thousands of connections on a handful of asset pairs, not hundreds of thousands of connections across thousands of pairs. Live data was underinvested and barely used in the consumer app.

I was on the team that owned GraphQL, and was the directly responsible individual for the frontend data layer library. As GraphQL took off, I took ownership of the live data story. I built the performance comparison, wrote the recommendation, and led the team that shipped the first version."

## 0:45 GraphQL live data (15s)

**Show:** the GraphQL live data systems table.

"In GraphQL, live data comes in two forms. Subscriptions are the typical one: a stream of events that updates the cache on the client. Deferred queries use the `@defer` directive to stream the slow parts of a query after an initial payload of fast fields. We focused on subscriptions first, but knew we'd need to support deferred queries eventually."

## 1:00 Problem and requirements (45s)

"The existing prices system ran over WebSockets. It had scaling and performance problems that meant prices only updated every 3 to 5 seconds, and WebSockets had a history of leaked connections and a buggy reconnect loop. Whatever protocol we picked for GraphQL would become the foundation for every live feature.

The key requirements: delivery is one-way, from server to client, so WebSockets' bidirectional channel gave us no advantage. We needed to scale to hundreds of thousands of connections, support React Native and web, improve event latency, and have a resilient reconnect loop."

## 1:45 Comparing the options (75s)

**Show:** the summary table.

"We compared the two options on four considerations.

Operational experience favored HTTP. The routing team found WebSockets painful to support, mainly because of leaked connections, and on the frontend the reconnect loop was buggy.

Platform support favored WebSockets. React Native didn't fully support fetch streaming at the time, and the open source polyfills were incomplete.

Performance was a tie. I built a test suite, and the differences were tiny. Network instability would be the bottleneck long before the protocol.

The frontend architecture favored HTTP, and this was the deciding factor for me. Our data layer was built on fetch. Adding WebSockets would have split it into separate unary and streaming halves, and deferred queries would need to operate on both sides. One protocol meant one abstraction."

## 3:00 Overcoming skepticism (30s)

"There was early skepticism that HTTP could scale for live data. The assumption was that clients can only hold a few HTTP connections open, which is a holdover from HTTP/1.1. We showed that limits did exist on some platforms, around 60 connections on iOS in our testing, but that was more than enough for our needs."

## 3:30 Decision and frontend architecture (45s)

**Show:** the frontend architecture diagram.

"So we chose HTTP streams and took on the platform work. We patched the React Native polyfill and contributed the fix upstream.

On the frontend, unary queries and subscriptions share one pipeline: the same fetch, the same response wrapper, and the same middleware for authorization, retries and observability. The only difference is the parser at the start and whether the result resolves a promise or emits to a Relay observable."

## 4:15 Backend architecture (30s)

**Show:** the backend architecture diagram.

"To migrate prices, we built a new live-prices service designed for user traffic at scale. It had a distributed Redis cache with a TTL policy, Redis pub/sub to fan updates out to API instances, and optimizations like connection dedupe. It streamed to GraphQL over gRPC, and GraphQL streamed to clients over HTTP."

## 4:45 Outcomes (30s)

"Price updates went from every 3 to 5 seconds to under 1 second, and we scaled to around 500,000 concurrent connections. HTTP's simpler failure modes gave us a more resilient reconnect loop, and we removed WebSockets from the client entirely.

The architecture bet paid off: when `@defer` landed later with GraphQL federation, it fit the same pipeline with minimal changes. By the time I left, there were 75+ subscriptions in the schema, and `@defer` was used on basically every high priority flow to reduce latency."

## 5:15 Downsides (30s)

"The main cost was the React Native platform work that we had to maintain and document. Any project built on top of React Native that wanted to use GraphQL would have to add the same polyfills. Over time there was a small maintenance burden as we discovered problems with the polyfill setup (non-critical) that had to be patched."

_Pause and hand over to questions._

## If asked

| Question                                                   | Where to go                                                                                   |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Why not use WebSockets' bidirectional channel?             | Requirements. Subscriptions are one-way, so it gave no advantage.                             |
| What were the performance test results in detail?          | Considerations: Performance                                                                   |
| How did you get fetch streaming working on React Native?   | Decision and first use cases. Patched the open source polyfill and contributed upstream.      |
| Why build a new service instead of fixing the old one?     | Backend architecture: Why a new service. No distributed cache, 3-5s latency, upstream load.   |
| What do connection dedupe and inverse currency pairs mean? | Backend architecture: Why a new service                                                       |
| How did `@defer` fit in?                                   | Outcomes, and Frontend architecture. Same pipeline, minimal changes.                          |
| Did teams adopt it?                                        | Outcomes. Dozens of use cases in year one, 75+ subscriptions, the Wallet team's Base rebuild. |
