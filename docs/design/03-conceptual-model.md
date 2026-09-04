# Conceptual model

## The four nouns

### Function

A function is a typed, finite invocation. It has:

- a stable semantic identity and version;
- a Facet-described argument record and result;
- execution semantics;
- capability and policy requirements;
- one or more available implementations/placements;
- structured progress and diagnostics;
- optional artifact outputs and receipts.

A function has no durable identity as a running entity once its invocation finishes.

### Actor

An actor is a persistently hosted entity. It has:

- a stable instance identity and generation;
- a stable actor type identity and version;
- reflected configuration;
- domain state;
- lifecycle and supervision;
- timers and external wakeups;
- service implementations;
- named publications;
- optional checkpoints.

“Actor” in this design means a persistent ownership and message boundary. It does not require every implementation to expose Erlang semantics or distribute transparently.

### Service

A service is a typed trait/interface. It has:

- a stable semantic identity and version;
- named methods;
- Facet-described request and result shapes;
- an error contract;
- method-level concurrency/cancellation/idempotency metadata;
- transport-independent clients and dispatch;
- optional streaming values represented through portable abstractions.

An actor may implement multiple services. A persistent non-actor component may also implement a service.

### Flow

A Flow is a declarative graph. It contains:

- actor instance specifications;
- source bindings to actor publications or named artifacts;
- function nodes;
- keyed map/flat-map/collect/join operations;
- targets;
- policy and placement overrides;
- source maps back to KDL/Rust/Starlark authoring locations.

Flow is desired computation, not a transcript of execution.

## Supporting nouns

### Contract type

A Rust type with Facet reflection plus explicit durable/remote semantic identity. Its Contract Shape can be placed in manifests and compared across implementations.

### Artifact

An immutable semantically described value owned by Artifactum. Content identity and provenance are separate.

### Collection

An immutable keyed mapping of logical item keys to artifact identities. The key is domain identity; the artifact ID is immutable value identity.

### Publication

A named actor output whose current value points to an immutable artifact or sealed collection snapshot. Advancing a publication is atomic.

### Node

A function requirement plus argument bindings and execution policy:

```text
Node = Function + Bindings + Policy
```

The function remains independently callable outside the graph.

### Target

A named desired output of a Flow. Making a target current may require zero or more function executions.

### Plan

The concrete work decision for a target at a given source/publication state: cache hits, required attempts, blockers, placement, and reasons.

### Run

One orchestration episode requested to make one or more targets current.

### Attempt

One execution of one action/function realization request.

### Realization

A successful binding of a canonical action identity to immutable outputs.

### Receipt

An immutable record of an observation or effect, including relevant external identity and attempt evidence. A receipt does not falsely imply that the external world can be replayed.

## Function semantics

The proposal distinguishes four user-visible forms.

### Derive

A derivation transforms immutable inputs. It maps to Artifactum's `pure` or `reproducible` cache semantics.

```rust
#[flow::derive(id = "auction.parse", version = 1, cache = "pure")]
async fn parse(page: Artifact<HtmlPage>) -> Result<ListingBatch, ParseError>;
```

### Function

A normal callable operation defaults to volatile execution. It may be used for search, inspection, ad hoc analysis, or computations whose key is intentionally not treated as reusable.

```rust
#[flow::function(id = "auction.search", version = 1)]
async fn search(query: SearchQuery) -> Result<SearchResults, SearchError>;
```

### Observe

An observation reads mutable external reality and produces immutable evidence and a receipt. It is finite even when invoked by a persistent actor.

```rust
#[flow::observe(id = "web.crawl-once", version = 1)]
async fn crawl_once(spec: WebsiteSpec) -> Result<CrawlObservation, CrawlError>;
```

### Effect

An effect mutates an external system. It always executes and must produce an immutable receipt or declared output.

```rust
#[flow::effect(id = "auction.send-alert", version = 1)]
async fn send_alert(request: AlertRequest) -> Result<AlertReceipt, AlertError>;
```

The distinction governs planning, caching, replay language, UI warnings, and policy admission.

## Why `observe`, not `source`, as a function kind

“Source” is overloaded in workflow systems. It can mean a connector, a graph root, mutable external data, or a source-code input. This design uses:

- **observe function** for one finite external read;
- **actor** for persistent source behavior;
- **publication** for the actor's named current output;
- **source binding** in Flow IR for bringing a publication/artifact into a graph.

This vocabulary makes time and ownership explicit.

## Actor-to-Flow relationship

Actors do not insert arbitrary graph nodes. They publish immutable source state:

```text
external site
    |
observe function
    |
actor domain policy and checkpoint
    |
open publication session
    |
immutable item artifacts
    |
seal collection snapshot
    |
advance named publication
    |
Flow source revision changes
    |
re-plan affected derivations
```

The actor decides *when to look*. Flow decides *what derived work is now stale*.

## Local and remote equality

“Same function/service” means the same semantic descriptor and contract, not the same implementation artifact or transport.

A call may resolve to:

- direct ordinary Rust call;
- dynamic linked invocation;
- persistent Remoc worker;
- structured JSON worker;
- remote service endpoint.

Observable behavior must respect the same request/result/error contract. Performance, isolation, cancellation granularity, and supported optional capabilities may differ and are advertised.

## Values versus resources

Boundary-safe values include:

- scalar/config records;
- semantic artifact references;
- collection references;
- actor/service references;
- secret references;
- portable stream/subscription references;
- explicit effect/observation receipts.

Live resources include:

- `spider::Website`;
- database pools;
- model sessions;
- browser contexts;
- open sockets;
- process handles;
- mutable index writers.

Live resources do not cross semantic boundaries. A reflected spec crosses; `Hydrate` creates the resource where it will live.

## Three graph views

The tooling should expose three distinct views, following the useful distinction made by build systems such as Buck2:

1. **Declared graph** — functions, bindings, actors, and targets in Flow IR.
2. **Current plan** — concrete actions, cache hits, blockers, and placements for the present source state.
3. **Execution trace** — what actually ran, emitted, retried, failed, and realized.

Conflating these views makes debugging substantially harder.
