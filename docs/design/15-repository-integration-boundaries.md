# Repository integration boundaries

## Objective

The new design should extract reusable contracts from existing repositories without turning them into circular dependencies or duplicating their strongest components.

## Outboard

### Retain

- executable naming/discovery and precedence;
- plugin/interface/capability manifests;
- SemVer compatibility filtering;
- one-shot and worker execution;
- progress/output/cancellation lifecycle;
- OS-string-safe argv support;
- management and doctor commands;
- conformance testing;
- process as stable ABI.

### Generalize

- add Facet-backed function/service/actor descriptors;
- advertise transport capabilities;
- support structured JSON v2 calls;
- add Remoc adapter and stream-generic sessions;
- separate semantic contract from transport codec;
- expose actor/service discovery and resolution;
- retain legacy argv interfaces as compatibility adapters.

### Must not own

- Flow graph semantics;
- Artifactum action identity;
- actor domain state;
- daemon lifecycle internals;
- source-specific scheduling.

## Artifactum

### Retain

- content-addressed storage and semantic artifacts;
- trees and keyed collections;
- ActionSpec/ActionKey;
- Attempt/Realization;
- pure/reproducible/volatile/effect semantics;
- outputs committed after successful validation;
- logs/checkpoints on failure;
- lineage, refs, leases, verification, and GC;
- executor boundaries and artifact-reference transfer.

### Integrate

- Flow function descriptors lower to Artifactum action specs;
- actor publications point to Artifactum artifacts/collections;
- open publication sessions may build on collection primitives;
- observation/effect functions use Artifactum receipts;
- `flow explain/replay/lineage` reuse Artifactum data.

### Must not own

- actor timers or service mailboxes;
- Flow KDL/IR;
- mutable scheduler lanes/queues beyond its own attempt engine;
- plugin discovery semantics already owned by Outboard.

Artifactum currently contains its own executable provider/executor plugin hosting. The long-term architectural goal should be convergence on Outboard's generalized contract/transport substrate rather than two unrelated plugin protocols. The exact migration is outside this design package.

## Daemonkit

### Retain

- stable local daemon identity;
- keyed/isolation-scoped instances;
- startup serialization and authenticated bootstrap;
- embedded and managed daemon shapes;
- readiness publication;
- generation-safe attachment;
- graceful shutdown/forced process-tree cleanup;
- stale-state repair;
- deterministic lifecycle testkit.

### Integrate

- `daemonkit-remoc` establishes Remoc over authenticated local streams;
- an Outboard persistent host may use Daemonkit for cross-CLI reuse;
- actor host generation changes map to explicit runtime events.

### Must not own

- Remoc or JSON framing in core Daemonkit;
- function/service descriptors;
- actor state or publication storage;
- workflow control queues.

## Switchyard

Switchyard has already explored a backend-neutral semantic service with static, native, process, shared-memory, and Wasm implementations. Its strongest reusable lessons are:

- preserve original service traits;
- generate backend-neutral clients and dispatch;
- publish generations rather than pretend arbitrary native Rust is safely unloadable;
- pin calls to generations;
- reject nonportable service types early;
- keep trusted native and isolated process threat models distinct;
- make build IDs and schemas explicit;
- test adversarial backends.

The new Outboard service macro should reuse design/code where appropriate, but Facet Contract Shape should replace Switchyard's stringified type-schema representation.

Switchyard remains useful as an optional linked/native/Wasm execution backend. It should not become mandatory for the normal process/Remoc/JSON workflow path.

## Auctions

### Existing lessons to preserve

- new sources may expand the domain model, but ordinary additions should not require crawler-architecture changes;
- two-stage cheap discovery and selective detail/image fetch;
- raw pages and observations retained;
- parsers are pure over fixtures;
- missing data is explicit;
- scoring reasons matter more than opaque scores;
- long-running scan/analysis service has a real queue and interactive field client;
- budgets and external model calls must be explicit;
- every listing and price observation builds a historical corpus.

### Expected boundary

Auctions supplies:

- domain contract types (`Listing`, observations, classifications, reports);
- source-specific actor configs/policies;
- parser/classifier/ranker functions;
- actor instances and KDL Flow definitions;
- application CLI commands and views.

The generic system supplies:

- actor lifecycle and service dispatch;
- Spider hydration/reference actor;
- artifact/collection publication;
- function registration/invocation;
- Flow planning and operational tooling;
- transport/placement.

## PartFoundry

### Existing lessons to preserve

- observations are evidence, not canonical identity;
- append-only raw/Bronze/Silver history;
- Artifactum owns source, Parquet, manifest, and document bytes;
- PostgreSQL owns semantic/control projections rather than bulk bytes;
- workflow owns scheduling/leases/retries/budgets, not derivation identity;
- source operations commit exact profile/policy/plugin/input identities;
- stale attempts cannot commit terminal success;
- source plugins are sandboxed Outboard executables;
- policy, authority, rights, retention, and downstream use remain distinct.

### Expected boundary

PartFoundry supplies:

- source/authority/policy contract types;
- manufacturer/source actors and services;
- acquisition/parse/reconcile/promote functions;
- Bronze/Silver/Gold domain rules;
- rights and evidence policies;
- application-specific web/CLI views.

The generic system supplies:

- actor/function/service descriptors;
- KDL/IR topology;
- transport/placement;
- Artifactum action integration;
- workflow control primitives;
- generic debugging/introspection.

The existing `partfoundry-workflow` is a concrete proving ground for control-plane invariants. Generic extraction should preserve its exact-contract enqueue, lease-token, stale-attempt, bounded retry, lane control, and paid-effect reservation behavior.

## Shared workflow control layer

A generic workflow control component may be extracted from PartFoundry, but its schema/API should remain focused:

```text
run requests
jobs/actions
attempt admission
leases/heartbeats
retry timing
lanes/priority
budget reservations
terminal event convergence
```

It should not absorb:

- Artifactum action/realization tables;
- actor domain state;
- arbitrary application records;
- generic outbox/event-sourcing abstractions;
- KDL definition storage as mutable queue state.

SQLite and PostgreSQL backends may both exist behind a narrow contract. Backend parity should be semantic, not schema-identical.

## Spider crate

Spider remains an external dependency and live crawler implementation. The generic integration owns only:

- stable reflected `WebsiteSpec` and associated policies;
- `Hydrate` adapter;
- actor/service wrapper;
- page artifact/publication contracts;
- capability mapping;
- conformance/reference example.

Spider's full internal `Website` API should not become the public Flow contract. New Spider versions can be adapted inside hydration/actor implementation.

## Remoc crate

Remoc remains an external transport implementation. The integration owns:

- adapter generation;
- session establishment through Outboard/Daemonkit;
- mapping of semantic streams/refs;
- capability advertisement;
- error/cancellation conversion;
- conformance tests.

Forking Remoc or copying its RPC system into Outboard is not the default design.

## Facet and Figue

Facet is the reflection authority inside Rust. Figue is a likely default CLI/config implementation. Our durable contract remains Contract Shape and our command/event protocol; this protects project APIs from direct dependence on one Facet ecosystem crate's private representation.

## Dependency direction summary

```text
Daemonkit          Artifactum          Facet
    ^                  ^                 ^
    |                  |                 |
Outboard core/contract/remoc/json       |
    ^                  ^                 |
    +------------------+-----------------+
                       |
                  Flow core/IR/runtime
                       |
            Auctions / PartFoundry / apps
```

No application repository should be required by a generic crate.
