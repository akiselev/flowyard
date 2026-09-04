# Proposed public API surface

This file is a compact review map. Signatures are illustrative.

## Contract types

```rust
use facet::Facet;
use flow::Contract;

#[derive(Facet, Contract)]
#[contract(id = "auction.listing", version = 2)]
pub struct Listing {
    pub source: SourceId,
    pub external_id: String,
    pub title: String,
}
```

Core traits/types:

```rust
trait Contract {
    const ID: ContractTypeId;
    const VERSION: ContractVersion;
    fn shape() -> &'static facet::Shape;
    fn contract_shape() -> &'static ContractShape;
}

struct ContractShape;
struct ContractValue;
struct ContractTypeId;
struct ContractVersion;
```

## Hydration

```rust
pub trait Hydrate: Facet<'static> + Sized {
    type Runtime: Send + 'static;

    async fn hydrate(
        self,
        context: &HydrateContext,
    ) -> Result<Self::Runtime, HydrateError>;
}
```

## Function macros

```rust
#[flow::derive(id = "...", version = 1, cache = "pure")]
#[flow::function(id = "...", version = 1)]
#[flow::observe(id = "...", version = 1)]
#[flow::effect(id = "...", version = 1)]
```

Recognized context injection:

```rust
&flow::Context
&flow::ObserveContext
&flow::EffectContext
```

Core function types:

```rust
struct FunctionDescriptor;
struct FunctionContract;
struct FunctionRequirement;
trait LocalFunctionInvoker;
struct FunctionRegistry;
struct Invocation;
struct InvocationEvents;
```

Semantic wrappers:

```rust
Artifact<T>
Collection<T>
Secret<T>
ServiceRef<S>
ActorRef<S>
RemoteStream<T>
Subscription<T>
```

## Services

```rust
#[outboard::service(id = "spider.crawler", version = 1)]
pub trait CrawlerService {
    async fn status(&self) -> Result<CrawlStatus, CrawlerError>;
    async fn refresh(&self, request: RefreshRequest)
        -> Result<CrawlReceipt, CrawlerError>;
}
```

Core service types:

```rust
struct ServiceDescriptor;
struct MethodDescriptor;
struct ServiceRequirement;
trait ServiceTransport;
struct ServiceClient<T>;
struct ServiceFault;
```

## Actors

```rust
#[outboard::actor(id = "spider.website", version = 1)]
pub struct SpiderCrawler { ... }

impl outboard::Actor for SpiderCrawler {
    type Config = WebsiteSpec;
    type State = CrawlerState;

    async fn create(...);
    async fn run(...);
}

#[outboard::service_impl]
impl CrawlerService for SpiderCrawler { ... }
```

Core actor types:

```rust
trait Actor;
struct ActorDescriptor;
struct ActorRequirement;
struct ActorInstanceSpec;
struct ActorContext;
struct ActorCreateContext;
struct ActorRef<S>;
struct ActorGeneration;
struct Publication<T>;
struct PublicationSession<T>;
```

## Registry and runtime

```rust
let registry = Registry::builder()
    .linked()
    .outboard(outboard_registry)
    .remote(remote_registry)
    .build()?;

let runtime = Runtime::builder()
    .registry(registry)
    .artifacts(artifactum)
    .control_store(control_store)
    .project(compiled_project)
    .build()?;
```

## Flow IR

```rust
struct ProjectIr;
struct ActorInstanceSpec;
struct FlowDefinition;
struct SourceBinding;
struct NodeSpec;
struct TargetSpec;
struct Binding;

enum NodeOperation {
    Call,
    Map { each: ArgumentName, over: Binding },
    FlatMap { each: ArgumentName, over: Binding },
    Collect,
    Join { mode: JoinMode },
}
```

Frontends:

```rust
flow_kdl::parse(source) -> CompiledProject
flow_ir::Builder
// future: flow_starlark::evaluate(source) -> CompiledProject
```

## Planning/execution

```rust
runtime.validate(targets)
runtime.graph(target)
runtime.plan(target)
runtime.run(target)
runtime.explain(subject)
runtime.replay(attempt)
runtime.fork(attempt, overrides)
```

## Events

```rust
enum EventKind {
    Started,
    Progress,
    Diagnostic,
    Log,
    ArtifactPublished,
    ItemPublished,
    Checkpoint,
    Finished,
    Failed,
}

struct EventEnvelope {
    protocol: EventProtocolVersion,
    request_id: RequestId,
    sequence: u64,
    time: Timestamp,
    subject: SubjectRef,
    event: EventKind,
    data: ContractValue,
}
```

## CLI embedding

```rust
let integration = flow_cli::Integration::builder(runtime)
    .mount_at("workflow")
    .enable_functions()
    .enable_actors()
    .enable_runs()
    .build();

application.mount(integration);
```

Domain projection:

```rust
flow_cli::commands()
    .function("auction.search").as_command("search")
    .actor_method("govdeals", "spider.crawler", "refresh")
        .as_command("scan");
```

## Outboard transport capabilities

```text
linked
outboard.transport.remoc-v1
outboard.transport.json-v2
outboard.transport.argv-v1
outboard.events-v1
outboard.artifact-ref-v1
```

## Stock commands

```text
flow catalog
flow function list|describe|call
flow actor list|describe|status|call|restart|stop|repair|publications
flow check|graph|plan|run|status|runs
flow explain|lineage|logs
flow attempt replay|fork|shell|materialize
flow repl
flow fmt|edit|doctor
```
