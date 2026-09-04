# Macros and generated code

## Role of macros

Macros should remove repetitive adapters, not hide the runtime model. Their responsibilities are:

- extract a stable semantic contract from ordinary Rust declarations;
- generate Facet-reflected request/argument records;
- generate thin linked/transport dispatch adapters;
- register descriptors;
- reject nonportable signatures with precise diagnostics;
- preserve original functions, traits, structs, and impls.

Scheduling, retries, persistence, artifact import, actor runtime, and transport state machines belong in normal crates, not generated token streams.

## Proposed macro surface

### Contract types

```rust
#[derive(Facet, Contract)]
#[contract(id = "auction.listing", version = 2)]
pub struct Listing {
    pub source: SourceId,
    pub external_id: String,
    pub title: String,
}
```

`Contract` supplies semantic identity and portable projection over the existing Facet shape.

### Functions

```rust
#[flow::derive(id = "auction.parse", version = 1, cache = "pure")]
async fn parse(...);

#[flow::function(id = "auction.search", version = 1)]
async fn search(...);

#[flow::observe(id = "web.crawl-once", version = 1)]
async fn crawl_once(...);

#[flow::effect(id = "auction.publish-alert", version = 1)]
async fn publish_alert(...);
```

Internally these may share one parser/expander and differ only in semantic defaults.

### Services

```rust
#[outboard::service(id = "spider.crawler", version = 1)]
pub trait CrawlerService {
    async fn status(&self) -> Result<CrawlStatus, CrawlerError>;
    async fn refresh(&self, request: RefreshRequest)
        -> Result<CrawlReceipt, CrawlerError>;
}
```

### Actors

```rust
#[outboard::actor(id = "spider.website", version = 1)]
pub struct SpiderCrawler { ... }

impl outboard::Actor for SpiderCrawler { ... }

#[outboard::service_impl]
impl CrawlerService for SpiderCrawler { ... }
```

The separate `service_impl` annotation lets one actor implement multiple services without putting a service list in the actor's attribute grammar.

## Function expansion model

Conceptual expansion:

```rust
async fn classify(
    context: &flow::Context,
    listing: Artifact<Listing>,
    threshold: f32,
) -> Result<Classification, ClassifyError> {
    // original body
}

#[doc(hidden)]
mod __flow_classify {
    use super::*;

    #[derive(Facet)]
    pub struct Args {
        pub listing: Artifact<Listing>,
        pub threshold: f32,
    }

    pub static DESCRIPTOR: FunctionDescriptor = /* static metadata */;

    pub struct LinkedInvoker;

    impl LocalFunctionInvoker for LinkedInvoker {
        fn invoke(&self, context: Context, args: OwnedValue)
            -> BoxFuture<'static, DynamicResult>
        {
            Box::pin(async move {
                let Args { listing, threshold } = args.materialize()?;
                let result = super::classify(&context, listing, threshold).await?;
                Ok(OwnedValue::new(result))
            })
        }
    }

    register_function!(DESCRIPTOR, LinkedInvoker);
}
```

The exact `OwnedValue` representation is intentionally not fixed by the public API.

## Context parameters

The function macro recognizes a limited set of injected parameters, normally first:

```rust
&flow::Context
&flow::ObserveContext
&flow::EffectContext
```

Injected context is omitted from the portable argument record. Other parameters are ordinary named inputs.

A context should expose capabilities explicitly and record accesses that affect identity/policy. It must not be an unbounded escape hatch to arbitrary global state.

## Parameter attributes

Proposed attributes include:

```rust
#[flow(default = 0.72)]
#[flow(rename = "minimum-confidence")]
#[flow(secret)]
#[flow(flatten)]
#[flow(help = "...")]
#[flow(example = "...")]
#[flow(output)]
```

Most field semantics should ultimately be represented on the generated Facet shape. The function macro translates parameter attributes into the hidden `Args` field attributes and descriptor metadata.

The attribute set should remain narrow. General validators live on types/specs rather than per-parameter macro strings.

## Signature restrictions

### Function boundary

Exported dynamic functions should reject:

- generic functions or unconstrained `impl Trait` inputs;
- borrowed input values other than recognized contexts;
- borrowed outputs;
- raw pointers and function pointers;
- process-local trait objects;
- unnamed/destructuring parameter patterns;
- types lacking Facet reflection;
- durable/remote types lacking semantic Contract identity where required;
- mutable references to caller-owned values;
- `unsafe`/`extern` signatures as portable functions.

A function may still use these internally through an ordinary wrapper.

### Service boundary

Portable service traits should reject:

- generic methods;
- associated types in the portable v1 profile;
- receiver by value unless explicitly modeled as terminal consumption;
- borrowed/raw-pointer boundary values;
- methods without a `Result`-like remote failure path;
- overloaded/duplicate wire names;
- incompatible default declarations.

Remoc supports some broader forms. The macro should not accept them by default merely because one transport can.

### Actor boundary

Actor types must define:

- a stable actor ID/version;
- a Facet-reflected config type;
- explicit state/checkpoint declarations;
- `Send + 'static` runtime behavior for the normal Tokio host;
- declared services/publications through generated registrations.

## Service expansion model

The service macro preserves the trait and generates a hidden module containing:

```text
SERVICE_ID / VERSION
ServiceDescriptor
one MethodDescriptor per method
one named Request type per method
backend-neutral Client<T: ServiceTransport>
dispatch(request)
JSON adapter
optional Remoc adapter
request-receiver/actor-mailbox adapter
conformance helpers
```

Method IDs should derive from explicit canonical service ID, major version, and stable method wire name using a collision-resistant digest/truncation scheme. Rust type spelling does not enter method identity.

## Request records and compatibility

Even a multi-argument method lowers to a named request record:

```rust
async fn search(&self, query: String, limit: u32) -> ...
```

becomes conceptually:

```rust
#[derive(Facet, Contract)]
struct SearchRequestWire {
    query: String,
    limit: u32,
}
```

This matches Remoc's useful named-argument compatibility model and makes JSON/CLI behavior coherent.

## Remoc code generation strategy

The semantic macro should own the source model. The Remoc adapter may be generated in one of two ways:

- generate a private Remoc remote trait mirroring the portable service; or
- generate conversion/dispatch glue around Remoc request channels.

The public trait remains transport-neutral. The adapter adds Serde/remote-send bounds only under the Remoc feature and emits a targeted compile error if a contract cannot support that transport.

The design must not require every service crate to expose Remoc-generated request enums as its public API.

## Actor service implementation expansion

`#[outboard::service_impl]` preserves the original impl and generates an adapter that:

- associates actor type + service descriptor;
- converts portable/Remoc/JSON requests into actor mailbox messages;
- returns responses through the corresponding channel;
- applies method concurrency/cancellation/deadline metadata;
- emits tracing and diagnostic context;
- registers the implementation in actor descriptors/manifests.

Business logic remains the ordinary trait impl.

## Registration

The public contract should support both:

```rust
Registry::linked()
```

and explicit registration:

```rust
Registry::builder()
    .function(__flow_classify::registration())
    .actor(SpiderCrawler::registration())
    .build();
```

Automatic distributed registration may use `inventory` or `linkme`; that choice must not leak into function/service APIs. Registries sort by stable identity and treat duplicate implementation registrations as explicit conflicts/alternatives, never linker-order precedence.

Explicit registration is required for platforms or applications that avoid linker-section collection.

## Macro crate architecture

The proc-macro crate should remain a thin shell over a testable internal model:

```text
flow-macros/
    lib.rs
    parse.rs
    model.rs
    validate.rs
    expand/
        function.rs
        contract.rs
        registration.rs

outboard-macros/
    lib.rs
    parse.rs
    model.rs
    validate.rs
    expand/
        service.rs
        actor.rs
        service_impl.rs
        remoc.rs
        json.rs

macro-support/
    public generated-code support and stable helper APIs
```

Parsing produces a small normalized model. Validation accumulates precise errors. Expansion consumes only validated models.

`darling` is worth using for attribute parsing where it improves accumulated, span-preserving diagnostics. Complex signature validation still requires direct `syn` analysis.

## Generated-code stability

Generated module names are implementation details. Downstream code should access documented descriptor/registration functions rather than hidden names.

Macros must use `$crate`/resolved crate paths robustly and support renamed dependencies. Generated code should avoid broad imports and surprising lint suppressions.

A separate macro-support crate is public only because downstream expansion must reference it; its API should be intentionally narrow and versioned.

## Diagnostics as API

Examples of required compile-time diagnostics:

```text
`auction.classify` parameter `listing` crosses a Flow boundary but
`Listing` has no semantic Contract ID.

help: derive `Contract` and add #[contract(id = "...", version = 1)]
```

```text
service method `watch` returns `&Status`; borrowed outputs cannot cross
linked/process/remote service boundaries.

help: return owned `Status` or `Arc`-independent `StatusSnapshot`
```

```text
`#[flow::derive(cache = "pure")]` requests network capability.

help: move the network read into an `observe` function or mark the derivation
reproducible with an explicit immutable source artifact
```

Errors should point to the parameter/type/attribute that caused the problem, not a generated trait-bound failure.

## Testing generated APIs

Macro quality requires:

- `trybuild` compile-pass/compile-fail tests for diagnostics;
- expansion snapshots for descriptor and adapter changes;
- Contract Shape snapshots;
- linked invocation tests;
- Remoc and JSON conformance tests using the same service;
- renamed-dependency tests;
- downstream-crate integration tests;
- rustdoc examples for the intended minimal path;
- fuzz/property tests for attribute parsers and schema projection where useful.

The existing Switchyard macro code is useful precedent, particularly preservation of the original trait and early rejection of references, raw pointers, generics, and unsupported associated items. Its use of stringified Rust type names should not be carried forward.

## Macro non-goals

Macros should not:

- rewrite calls between annotated functions;
- infer a Flow graph from arbitrary Rust control flow;
- generate database schemas/migrations per function;
- hide network or secret access;
- automatically make a type cross-language safe merely because it implements Serde;
- generate a bespoke CLI parser unrelated to Facet;
- embed application-specific retry or scheduling logic.
