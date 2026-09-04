# Executive summary

## Proposal

Build a manifest-first workflow and callable-component layer with two equally supported deployment styles:

- a stock standalone `flow` CLI that discovers external executables through Outboard; and
- an embeddable CLI/runtime library that lets applications link Rust functions and actors directly.

The semantic graph is stored as versioned **Flow IR**. KDL v2 is the default human-authored frontend. Rust and a future Starlark frontend may produce the same IR, but they receive no privileged runtime semantics.

The system has four public computational concepts:

| Concept | Meaning |
|---|---|
| Function | A typed, normally finite operation. It may be a deterministic derivation, a volatile operation, an external observation, or an effect. |
| Actor | A persistently hosted component with identity, configuration, state, lifecycle, autonomous wakeups, services, and named publications. |
| Service | A typed trait/interface exposed by an actor or another persistent implementation. |
| Flow | A declarative graph that binds function arguments to literals, artifacts, collections, and actor publications. |

A graph node is not a special implementation type:

```text
Function + bindings + execution policy = node
```

This keeps application authors out of scheduler APIs.

## Reflection and contracts

Rust values derive [`Facet`](https://github.com/facet-rs/facet). Facet's `Shape` is the in-process source of truth for fields, variants, documentation, attributes, defaults, and construction. A deterministic **Contract Shape** is projected from Facet for manifests, schema fingerprints, KDL validation, JSON interoperation, documentation, and agent tool discovery.

Contract Shape is deliberately not a Rust ABI. It excludes layout, pointers, vtables, compiler `TypeId`, source paths, and other process-specific details. Types that cross durable or remote boundaries have an explicit semantic ID and major version.

Configuration/specification types may implement `Hydrate` to construct live runtime objects:

```rust
pub trait Hydrate: Facet<'static> + Sized {
    type Runtime: Send + 'static;

    async fn hydrate(
        self,
        context: &HydrateContext,
    ) -> Result<Self::Runtime, HydrateError>;
}
```

This is the generalized replacement for trying to serialize live values such as `spider::Website`, database pools, loaded models, browser sessions, or service clients.

## Functions

An ordinary Rust function remains ordinary Rust:

```rust
#[flow::derive(id = "auction.parse", version = 1, cache = "pure")]
async fn parse(
    context: &flow::Context,
    page: Artifact<HtmlPage>,
) -> Result<Collection<Listing>, ParseError> {
    // ordinary Rust
}
```

The macro preserves the function and generates a hidden Facet-reflected argument record, descriptor, linked invoker, diagnostics, and registration metadata. Direct Rust calls incur no framework overhead. Dynamic linked calls use type erasure but no wire serialization. External calls use the same descriptor through Outboard.

The proposed function kinds are:

- `derive`: cacheable computation over immutable inputs;
- `function`: volatile or ad hoc operation;
- `observe`: one finite observation of mutable external reality;
- `effect`: an externally visible mutation that always produces a receipt.

Actors normally decide when to invoke observation functions. This avoids embedding refresh policy in the graph scheduler.

## Actors and services

Actors are persistent ownership and lifecycle boundaries, not merely long-running tasks. They may:

- retain small control state and references to bulk checkpoint artifacts;
- expose one or more typed services;
- wake on timers, manual calls, publication changes, or external signals;
- invoke functions;
- atomically publish named immutable artifacts or keyed collection snapshots.

Service traits are transport-independent. The recommended Rust transport is Remoc because it provides generated remote traits, cancellation, pipelining, remote objects/channels, and a request-receiver mode suitable for actor mailboxes. Remoc is an adapter, not the semantic contract: its Serde and Rust constraints must not leak into Contract Shape.

Outboard remains the discovery, manifest, capability, process, and placement layer. Its JSON worker protocol is retained as the language-neutral fallback and evolves to accept structured function and service calls in addition to legacy argv-oriented invocation. Large data crosses boundaries as Artifactum references or materialized files, not large JSON messages.

Preferred execution is policy-driven, typically:

```text
linked implementation
    > Remoc persistent worker
    > structured Outboard JSON worker
    > legacy one-shot/argv adapter
```

This is a preference, not a correctness hierarchy. Isolation, trust, platform, and placement policy may require a lower option.

## Flow IR and KDL

Flow IR is the canonical, versioned representation. It contains actor bindings, function requirements, typed argument bindings, graph operators, targets, and policy overrides. Source spans and comments are stored separately so formatting changes do not alter graph identity.

The initial graph algebra remains small:

```text
source / call / map / flat_map / collect / join / target
```

KDL maps naturally onto this model and its Rust parser preserves formatting and comments. KDL deliberately has no loops, conditionals, templates, or arbitrary expressions. Repetitive programmatic graph generation belongs in Rust or an optional Starlark frontend that emits Flow IR.

## Storage and execution ownership

- **Artifactum** owns immutable artifact content, collections, semantic manifests, action identities, attempts, realizations, cache semantics, lineage, and bulk checkpoints.
- **Workflow control storage** owns desired runs, readiness, leases, retries, priorities, admission, and references to Artifactum outputs.
- **Outboard** owns executable discovery, implementation manifests, compatibility, process boundaries, and transport negotiation.
- **Daemonkit** owns local daemon lifecycle, authenticated local endpoints, generation-safe attachment, readiness, restart, and stale-state repair.
- **Actors** own domain-specific persistent control state and publication policy; they do not appropriate Daemonkit state or Artifactum identity.

## Interactive and agent-facing operation

The stock CLI is a thin use of an embeddable `flow-cli` library. Applications may mount its commands under their own CLI, expose selected functions as domain commands, and register linked implementations.

The primary interactive interface is line-oriented rather than an alternate-screen TUI. Every REPL action has a one-shot equivalent. Required machine modes are stable JSON and JSONL, with request IDs and structured events. This matches how Codex and Claude Code operate in non-interactive/headless modes.

A richer inline TUI may add completion, tables, and progress when attached to a terminal, but it is only a renderer. It must never be the sole interface.

## Canonical example

A Spider crawler exercises the entire design:

```text
WebsiteSpec --Hydrate--> spider::Website
                         |
                   SpiderCrawler actor
                         |
              Crawler service: status/refresh/pause
                         |
        open keyed publication of immutable HtmlPage artifacts
                         |
                    seal snapshot
                         |
            Flow map(parse) -> map(classify) -> target
```

Pages may trigger per-item downstream work while a crawl is open, but a named durable collection publication advances atomically only when sealed. A failed crawl may leave reusable page artifacts and per-item realizations without presenting a partial snapshot as complete.

## Principal risks

1. Macro scope can grow uncontrolled. Generated code must be thin adapters over testable runtime crates.
2. Facet is active and broad. Durable contracts must depend on our Contract Shape projection, not Facet's internal memory representation.
3. A rich Remoc backend may tempt the public API to become Rust-only. Portable service semantics must remain enforceable.
4. Actors can become a second scheduler. They should own refresh/domain state, while Flow owns derivation planning.
5. Streaming publication can compromise snapshot correctness. Open sessions and atomic sealing must be explicit.
6. A flexible KDL decoder can become ambiguous. The frontend should favor one canonical spelling and excellent diagnostics.

## Result

The proposed architecture gives Auctions and PartFoundry a shared ingestion/derivation substrate without forcing them into a single binary or language. Rust applications can compile hot functions and actors directly; external implementations remain first-class; agents get a discoverable typed command surface; and the system reuses the strongest existing boundaries instead of rebuilding storage, lifecycle, or process isolation.
