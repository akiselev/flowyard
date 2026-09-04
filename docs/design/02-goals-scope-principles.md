# Goals, scope, and principles

## Problem statement

Auctions and PartFoundry independently accumulated the same classes of infrastructure:

- acquiring mutable external data;
- retaining exact raw evidence;
- parsing and normalizing into typed records;
- mapping over independently keyed items;
- conditionally promoting expensive processing;
- retrying, budgeting, and recovering unattended work;
- debugging source changes and failed runs;
- calling external implementations without linking every language/runtime;
- exposing operational state to humans and agents.

The proposed system extracts this shared shape without turning either application into a generic framework. The measure of success is not abstraction count. It is how little application-specific plumbing remains after both real workloads use the design.

## Primary goal: human and agent developer experience

A normal new transform should look like:

```rust
#[flow::derive(id = "auction.normalize", version = 1, cache = "pure")]
async fn normalize(
    context: &flow::Context,
    raw: Artifact<RawListing>,
) -> Result<Listing, NormalizeError> {
    // domain logic
}
```

and a graph use should be only the binding:

```kdl
map "normalize" use="auction.normalize@^1" {
    each "raw" from="parse.listings"
}
```

An author or coding agent should not need to edit:

- a scheduler registry;
- a database migration;
- retry loops;
- worker process framing;
- artifact import code;
- cache-key code;
- a separate CLI definition;
- a separate RPC schema;
- a dashboard route;
- a source-specific daemon supervisor.

## Secondary goals

### Rust-native performance without Rust-only architecture

Applications must be able to link heavy Rust functions directly and avoid process/RPC serialization. The same semantic functions must remain externally implementable through Outboard.

### Incremental, artifact-centered execution

The system should answer “what result is current?” rather than merely “did a job run?” Immutable artifacts and keyed collections are the dependency currency. Runs and attempts are operational history.

### Persistent source intelligence

A source component should be able to remember checkpoints, reason about freshness, back off, respond to manual refresh, and publish new immutable snapshots. This belongs in actors rather than generic cron syntax embedded in a DAG.

### Introspection before execution

Users and agents should be able to discover functions, actors, services, types, graph structure, compatibility, placement, and required permissions without running domain code.

### Reproducible debugging

A failed or surprising result should retain enough identity to reconstruct its inputs, implementation, parameters, environment, policy, logs, checkpoints, and lineage.

### Cross-language implementations

Rust is the best-supported authoring language, not the only implementation language. Python/Node/JVM executables should receive equivalent descriptors, structured calls, events, and artifact references.

## Non-goals

The design does not attempt to provide:

- a general distributed cluster scheduler in the first architecture;
- a full Erlang-style distributed actor system;
- deterministic replay of arbitrary workflow source code in the Temporal sense;
- exactly-once external effects;
- a replacement for Spider, Artifactum, Outboard, Daemonkit, Remoc, or a database;
- a universal ETL expression language;
- arbitrary loops and conditionals in KDL;
- transparent serialization of live runtime objects;
- a stable Rust dynamic-library ABI;
- automatic package installation or a public plugin registry;
- zero-copy for every placement and language combination;
- a generic UI framework.

## Design principles

### 1. One concept owns one kind of state

Artifactum owns immutable computational artifacts and provenance. The workflow control store owns mutable orchestration state. Daemonkit owns daemon lifecycle state. Actors own domain state. Outboard owns implementation discovery and process compatibility.

### 2. Values cross boundaries; live resources are hydrated

A stable specification may cross a boundary. A live `Website`, pool, browser, model, socket, or actor instance does not. Such values are created from reflected specifications inside an authorized context.

### 3. The executable boundary is the portability boundary

Rust linking is an optimization and integration mode. Process execution remains the stable ABI for independently shipped implementations.

### 4. The graph describes topology, not algorithms

Flow IR connects functions, collections, actor publications, and targets. Parsing, filtering, retry classification, source-specific refresh logic, and domain algorithms remain in code.

### 5. Explicit semantics beat inference

Pure derivation, mutable observation, and external effect must not look identical. Actor publications, collection seals, semantic versions, and capability requirements are explicit.

### 6. The machine interface is not a screen scraper

Human rendering and machine rendering consume the same structured events and descriptors. JSON/JSONL is stable. ANSI output and cursor movement are optional.

### 7. Errors should point to the caller's abstraction

A KDL error should identify the source span, function argument, expected Contract Shape, and available correction. A transport failure should identify placement and transport. A cache miss should explain the exact identity component that changed.

### 8. Preserve direct use

A macro-annotated function remains directly callable and unit-testable. A service trait remains a normal trait. An actor struct remains an ordinary type. The framework adds adapters rather than replacing normal Rust.

### 9. Optimize common paths, not hypothetical universality

The design must first make Auctions and PartFoundry substantially simpler. Escape hatches exist, but they do not dominate the surface.

### 10. Dynamic behavior should have an explicit boundary

Static KDL compiles to static Flow IR. Dynamic graph generation, where eventually needed, is a separate planner/frontend capability rather than implicit runtime mutation hidden inside ordinary transforms.

## Success criteria for the design

The architecture is credible when a reviewer can trace these cases without inventing missing concepts:

1. a linked pure Rust transform;
2. the same transform as a persistent external Rust worker over Remoc;
3. an equivalent Python implementation over JSON;
4. a Spider actor that periodically refreshes and publishes keyed page snapshots;
5. a downstream map that reuses unchanged items;
6. a manually callable service method;
7. a custom application CLI with selected Flow commands;
8. a coding agent that discovers, invokes, observes, and debugs everything through non-interactive commands.
