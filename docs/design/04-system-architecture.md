# System architecture

## Architectural overview

```text
            Rust / KDL / future Starlark authoring
                           |
                           v
                 Facet-reflected contracts
                           |
                   Flow IR + source map
                           |
                           v
                validation and resolution
                           |
             +-------------+--------------+
             |                            |
             v                            v
        actor runtime                 flow planner
             |                            |
      services/publications          concrete actions
             |                            |
             +-------------+--------------+
                           |
                    execution fabric
       +-------------------+-------------------+
       |                   |                   |
     linked              Remoc               JSON
       |             persistent Rust     cross-language
       +-------------------+-------------------+
                           |
                 Artifactum execution/data
                           |
            immutable artifacts and lineage

   Outboard: discovery, manifests, compatibility, process placement
   Daemonkit: authenticated local daemon lifecycle and generations
   Control store: runs, readiness, leases, retries, admission, priority
```

## Architectural planes

### Contract plane

The contract plane contains:

- Facet `Shape` values for in-process reflection;
- explicit semantic type/function/service/actor IDs and versions;
- deterministic Contract Shape projections;
- descriptors and schema fingerprints;
- capability and policy declarations.

It answers: **what can be called, with what values, and what does it produce?**

### Definition plane

The definition plane contains:

- Flow IR;
- actor instance specifications;
- KDL source maps;
- target definitions;
- function and service requirements;
- placement/policy overrides.

It answers: **what topology does the project want?**

### Implementation plane

The implementation plane contains:

- linked function/actor registrations;
- Outboard executable candidates;
- plugin manifests;
- implementation build identities;
- supported transports and capabilities;
- health/doctor results.

It answers: **which concrete implementation can satisfy a semantic requirement?**

### Control plane

The control plane contains mutable operational state:

- target requests and runs;
- node readiness;
- attempt leases and heartbeats;
- retries and backoff;
- lane and global admission controls;
- cost/resource reservations;
- actor desired/running generations;
- references to Artifactum outputs and checkpoints.

It answers: **what work is admitted, running, blocked, retryable, or complete?**

### Data and provenance plane

Artifactum owns:

- exact bytes and directory trees;
- artifact semantic manifests;
- keyed collections;
- action specifications and keys;
- attempts and realizations;
- logs and checkpoints;
- source/effect receipts;
- lineage, verification, refs, leases, and garbage collection.

It answers: **what immutable values exist, how were they produced, and can they be reused?**

### Lifecycle plane

Daemonkit owns:

- local instance identity;
- startup serialization;
- transactional spawn/readiness/commit;
- authenticated local endpoints;
- generation-safe attachment;
- graceful shutdown and process-tree cleanup;
- stale-state repair;
- lifecycle diagnostics.

It answers: **is the local service process genuinely ours, compatible, and ready?**

### Interaction plane

The interaction plane provides:

- stock and embedded CLI commands;
- line-oriented REPL;
- JSON/JSONL request and event modes;
- optional inline TUI renderer;
- generated help, completion, and schemas;
- future MCP/HTTP adapters.

It answers: **how do humans and agents discover, invoke, and debug the system?**

## Ownership boundaries

### Artifactum is not the scheduler UI

Artifactum's Action/Attempt/Realization model is retained intact. Flow may translate a function invocation into an Artifactum `ActionSpec`, but it should not create a competing action identity or cache database.

The Flow control store may reference an Artifactum action key, attempt, realization, artifact, or checkpoint. It should not copy their canonical data.

### The workflow control store is not the artifact store

Operational queues and leases are mutable, high-churn state. Immutable content and provenance are append-oriented and graph-reachable. Combining them would complicate transactions, cache reuse, and garbage collection.

PartFoundry's current workflow boundary already demonstrates the useful split: the workflow layer owns scheduling, attempts, leases, retries, lane controls, and paid-effect budget accounting, while callers retain exact Artifactum output bindings.

### Actors are not Daemonkit daemons

An actor may be hosted in a Daemonkit-owned process, but the identities are different:

```text
Daemon instance: process/lifecycle identity
Actor instance: domain component identity
Actor generation: hosted state/service generation
```

One daemon may host one actor, many actors, or a registry. Daemonkit should remain unaware of actor messages and domain state.

### Remoc is not the interface definition language

Remoc is a high-value Rust transport and code-generation backend. Service compatibility is defined by the semantic descriptor and Contract Shape, not Remoc's generated Rust request types or codec.

### KDL is not the runtime model

KDL is one source representation. It compiles into Flow IR. Runtime decisions and artifact keys must never depend on KDL whitespace, comments, ordering where semantically irrelevant, or parser-specific syntax objects.

## Runtime configurations

### Standalone dynamic configuration

```text
stock flow CLI
    |
Outboard registry
    |
external functions and actors
    |
Remoc or JSON workers
```

This mode requires no application-specific rebuild when adding or replacing external implementations.

### Custom linked application

```text
application CLI
    |
flow-cli library
    |
linked registry + Outboard registry
    |
linked hot functions / actors, external optional functions
```

This mode allows direct invocation and in-process resources while retaining the same KDL/Flow IR and operational tooling.

### Mixed placement

A single graph can use:

- a linked parser;
- a local Remoc-backed model actor;
- a Python JSON enrichment function;
- an SSH/remote Artifactum executor;
- a remote service endpoint.

Placement is resolved per function/actor requirement and policy, not globally for the process.

## Logical crate boundaries

The exact package names are provisional, but the dependency direction should resemble:

```text
outboard-core
    discovery, identity, manifests, requirements, placement-neutral contracts

outboard-contract
    Facet descriptors, Contract Shape, function/service/actor descriptors

outboard-json
    structured language-neutral invocation and event protocol

outboard-remoc
    generated/adapted Rust service and function transport

daemonkit-remoc
    Remoc sessions over Daemonkit-authenticated local streams

flow-core
    function semantics, artifact wrappers, contexts, registry interfaces

flow-ir
    canonical graph representation and validation types

flow-kdl
    KDL parse/edit/source-map frontend

flow-runtime
    actor runtime, planning integration, execution coordination

flow-cli
    parser-neutral command model, dispatch, render events

flow-cli-figue / flow-cli-clap (possible adapters)
    concrete CLI integration

flow-macros / outboard-macros
    thin code generation over support crates

flow-spider
    WebsiteSpec hydration, crawler actor, contracts, reference integration
```

The dependency rule is strict:

- Outboard must not depend on Flow.
- Daemonkit must not depend on Outboard, Remoc, or Flow.
- Artifactum must not depend on an application-specific Flow frontend.
- Flow may depend on the semantic Outboard contract layer and Artifactum APIs.
- CLI/TUI crates depend on core descriptors, never the reverse.

## Registry composition

A runtime registry is a merged view, not one global static table:

```text
explicit registrations
    > linked distributed registrations
    > explicit Outboard overrides
    > project plugin roots
    > environment plugin roots
    > PATH discovery
    > configured remote registries
```

Duplicate semantic implementations remain visible. Resolution records which candidate won and why. Shadowed or incompatible candidates are introspectable.

## Policy evaluation points

Policy is evaluated at multiple distinct boundaries:

1. **Definition validation:** is the requested capability legal in this project?
2. **Implementation resolution:** is this implementation trusted and compatible?
3. **Hydration:** may this spec resolve secrets, network, files, or local services?
4. **Attempt admission:** are budget, lane, concurrency, and resource limits available?
5. **Execution:** enforce sandbox/network/time/cancellation constraints.
6. **Publication/promotion:** may this output advance a named ref or external effect?

A single `allowed: bool` flag cannot represent these decisions safely.

## Architectural invariants

1. A semantic contract never relies on Rust memory layout.
2. A process/remote implementation cannot choose Artifactum identity for returned bytes.
3. A named actor publication points only to an immutable artifact or sealed collection.
4. A partial/open publication cannot masquerade as a complete snapshot.
5. A successful realization is distinct from a successful actor service call unless the call explicitly invokes an Artifactum action.
6. Transport fallback never silently weakens required security, cancellation, streaming, or type capabilities.
7. Linked execution may optimize transport, but it does not change semantic function identity.
8. Source spans and human comments do not enter canonical Flow IR identity.
9. External effects are recorded with receipts and idempotency information, not claimed to be exactly once.
10. Every human-facing interactive operation has a non-interactive structured equivalent.
