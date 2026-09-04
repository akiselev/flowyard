# Services and actors

## Services

A service is a persistent typed interface. Proposed authoring:

```rust
#[outboard::service(id = "spider.crawler", version = 1)]
pub trait CrawlerService {
    async fn status(&self) -> Result<CrawlStatus, CrawlerError>;

    async fn refresh(
        &self,
        request: RefreshRequest,
    ) -> Result<CrawlReceipt, CrawlerError>;

    async fn pause(&self) -> Result<(), CrawlerError>;
    async fn resume(&self) -> Result<(), CrawlerError>;
}
```

The trait remains an ordinary Rust trait. The macro generates descriptors, clients, dispatch adapters, and transport modules.

## Portable service boundary

Portable v1 service methods SHOULD use:

- owned request and result values;
- explicit contract types;
- `Result<Output, Error>`;
- no borrowed output or raw pointer;
- no generic method;
- no compiler-layout-dependent type;
- no implicit ambient resource;
- stable method names.

Although Remoc can support more Rust features, the baseline interface must remain implementable through structured JSON.

### One request object versus many parameters

Both forms may be accepted:

```rust
async fn search(&self, query: String, limit: u32) -> Result<Results, Error>;
```

The macro still lowers this to a named request object. For public APIs expected to evolve, an explicit request type is preferred:

```rust
async fn search(&self, request: SearchRequest) -> Result<SearchResults, Error>;
```

This produces clearer compatibility reviews, defaults, documentation, and cross-language SDKs.

## Generated service surface

A service macro should produce, conceptually:

- `ServiceDescriptor` and `MethodDescriptor` values;
- stable service/method IDs;
- Facet request/result shapes;
- portable Contract Shape projection;
- backend-neutral typed `Client<T: ServiceTransport>`;
- request dispatcher;
- JSON call adapter;
- optional Remoc client/server/request-receiver adapter;
- CLI method descriptions and argument schemas;
- conformance-test helpers.

This generalizes the useful shape already proven by Switchyard's `#[service]`: preserve the trait, generate a typed client, and generate dispatch. Facet replaces stringified Rust type schemas.

## Actors

Proposed actor authoring:

```rust
#[outboard::actor(id = "spider.website", version = 1)]
pub struct SpiderCrawler {
    website: spider::website::Website,
    state: SpiderCrawlerState,
}

impl outboard::Actor for SpiderCrawler {
    type Config = WebsiteSpec;
    type State = SpiderCrawlerState;

    async fn create(
        config: Self::Config,
        context: &ActorCreateContext,
    ) -> Result<Self, ActorError> {
        // hydrate Website, restore state/checkpoints
    }

    async fn run(
        &mut self,
        context: &mut ActorContext,
    ) -> Result<(), ActorError> {
        // actor-owned wakeup/refresh loop
    }
}

#[outboard::service_impl]
impl CrawlerService for SpiderCrawler {
    // normal implementation
}
```

The exact trait shape is illustrative; the semantic requirements are normative.

## Actor identity

An actor reference contains at least:

```text
actor type ID/version
instance key
host/placement identity
current or pinned generation
service set
```

Call policy determines whether a reference:

- pins one generation and fails after replacement;
- follows the current compatible generation;
- re-resolves placement after failure.

Generation behavior must be explicit because silent retargeting can violate stateful assumptions.

## Actor lifecycle

Actor lifecycle states should include:

```text
defined
hydrating
starting
ready
quiescing
stopped
failed
restarting
incompatible
```

Daemonkit can own the host process lifecycle. The actor runtime owns the actor lifecycle inside that process.

Lifecycle hooks may include:

- create/hydrate;
- restore state/checkpoint;
- started/readiness;
- run/wakeup loop;
- quiesce;
- checkpoint;
- stop;
- failure/supervision notification.

The API should favor a small number of hooks and derive common behavior from context rather than expose every scheduler event.

## Actor concurrency model

### Default: serialized actor domain

An actor's mutable domain state has one serialization domain. Service calls, timers, and internal wakeups are delivered as messages/events and ordered by the actor runtime.

This avoids accidental concurrent mutation and maps cleanly to Remoc's generated request-receiver mode, where requests can be processed as messages rather than directly dispatched onto an `Arc<RwLock<T>>`.

### Explicit concurrent reads

A service method may opt into concurrent read execution only when the actor exposes immutable/snapshot-safe state and the behavior is useful:

```rust
#[outboard(concurrency = "read")]
async fn status(&self) -> Result<CrawlStatus, Error>;
```

The default should not infer safety merely from `&self`, because interior mutability and consistency requirements remain possible.

### Long-running operations

An actor should not block its mailbox while performing a long network crawl if it needs to remain responsive. The normal pattern is:

1. actor admits and records an operation;
2. spawned task performs external I/O with an immutable operation spec;
3. task sends progress/completion back;
4. actor commits state/publication changes serially.

Generated actor plumbing may help, but business-specific operation splitting remains explicit code.

## Actor services and autonomous behavior

The same action can be triggered by:

- actor timer/staleness policy;
- manual service call;
- startup recovery;
- publication dependency change;
- external signal;
- operator command.

All triggers should converge on one domain method/operation identity so manual refresh does not implement a second code path.

Example:

```text
TimerExpired ----+
ManualRefresh ---+--> BeginRefresh(operation key) --> crawl task
StartupResume ---+
```

## Actor state

Actor state falls into three classes.

### Ephemeral runtime state

Open sockets, clients, in-flight tasks, and caches. Recreated by hydration; never treated as durable state.

### Small durable control state

Cursors, last successful observation, backoff, next wake, generation metadata, and publication refs. Stored transactionally by the actor runtime/control store.

### Bulk/recoverable state

Crawler frontier snapshots, model indexes, large checkpoints, downloaded partials, or browser state. Stored as Artifactum artifacts or implementation-owned externally referenced state with receipts.

Daemonkit private lifecycle state MUST NOT become the actor state database.

## Publications

An actor declares named typed publications:

```text
spider.website instance govdeals
    publications:
      pages   Collection<HtmlPage>
      health  CrawlHealth
      receipt CrawlReceipt
```

Publications are discoverable in the actor descriptor and callable tooling.

### Atomic advancement

A named publication points only to an immutable artifact or sealed collection. Advancement is atomic and generation-stamped.

### Open collection session

For streaming ingestion, an actor may open a publication session:

```rust
let mut publication = context
    .collection::<HtmlPage>("pages")
    .begin_from_previous()
    .await?;

publication.upsert(key, artifact).await?;
publication.checkpoint(cursor).await?;
publication.seal(metadata).await?;
```

During an open session:

- item artifacts are already immutable and durable;
- per-item events may trigger downstream mapped work;
- the previous sealed publication remains current;
- downstream aggregate targets wait for seal;
- abort/crash does not advance the named publication;
- completed per-item realizations remain reusable on retry.

This reconciles low-latency processing with snapshot correctness.

## Supervision

Actor supervision should provide:

- parent/host ownership;
- startup and terminal failure notification;
- restart policy with backoff/budget;
- generation transition records;
- health/readiness distinction;
- child task cleanup;
- operator-visible failure reason.

The design should learn from actor frameworks such as Ractor, while not adopting a separate cluster protocol when Outboard, Remoc, and Daemonkit already define the surrounding substrate.

## Actor references as function arguments

A function may accept a service/actor reference when persistent state is genuinely part of the computation:

```rust
async fn value_item(
    pricing: ServiceRef<PricingService>,
    listing: Artifact<Listing>,
) -> Result<Valuation, Error>;
```

The reference identity and relevant service generation/contract enter the action identity where they can affect output. If the service is nondeterministic or mutable, the function cannot truthfully claim pure semantics without an explicit snapshot/version token.

## Remote objects and streams

Remoc supports remote objects and channels. The portable contract should expose semantic wrappers instead of raw Remoc types:

```rust
RemoteStream<T>
Subscription<T>
ServiceRef<S>
ActorRef<S>
```

Adapters map these to:

- Remoc channels/remote clients in Rust;
- request IDs plus event frames in JSON;
- direct channels/handles when linked.

A method that explicitly requires Remoc-only semantics may declare a transport capability, but then it has no automatic JSON fallback. That exception should be visible in the descriptor and uncommon in baseline services.

## Actor anti-patterns

Actors should not become:

- generic queues for every function invocation;
- hidden mutable global singletons;
- owners of immutable artifact bytes;
- replacements for a scheduler;
- places where graph topology is constructed imperatively;
- silent ambient dependency injection;
- remotely migratable objects without explicit state/version semantics.
