# Prior art and research conclusions

## Research method

This review prioritized current primary sources: official project documentation and repository source, plus the existing Auctions, PartFoundry, Artifactum, Outboard, Daemonkit, and Switchyard repositories. The snapshot date is 2026-09-04.

No existing Rust workflow crate appears to provide the exact combination required here: durable artifact-centered incrementality, external executable implementations, direct linked Rust calls, persistent actors, transport-neutral remote traits, reflected CLI/config, and agent-oriented introspection. The design should therefore compose proven ideas rather than select one framework wholesale.

## Existing internal repositories

### Auctions

Auctions demonstrates the practical ingestion pipeline:

```text
discover -> retain raw evidence -> normalize -> score -> selective expensive enrichment -> rank/report
```

Key lessons:

- source adapters need a stable acquisition boundary;
- two-stage acquisition is economically important;
- keyed observations and historical prices matter;
- fixture-driven pure parsers isolate site changes;
- reasons/provenance are product behavior, not optional logs;
- image/model work needs budgets and promotion gates;
- a system designed only as a cron job eventually needs interactive operation and debugging.

References: [Auctions README](https://github.com/akiselev/auctions/blob/master/README.md), [Auctions design](https://github.com/akiselev/auctions/blob/master/DESIGN.md).

### PartFoundry

PartFoundry demonstrates stricter evidence and operational boundaries:

- observations are not automatically canonical facts;
- raw/Bronze/Silver data remains immutable;
- Artifactum owns bulk bytes and exact artifacts;
- PostgreSQL owns control/semantic projections;
- Outboard plugins isolate acquisition/parsing implementations;
- workflow leases and stale-attempt rejection are separate from derivation identity;
- rights, authority, retention, and downstream use are separately versioned decisions.

References: [PartFoundry README](https://github.com/akiselev/foundry/blob/master/README.md), [workflow implementation](https://github.com/akiselev/foundry/blob/master/crates/partfoundry-workflow/src/lib.rs), [implementation invariants](https://github.com/akiselev/foundry/blob/master/docs/notes/IMPLEMENTATION_INVARIANTS.md).

### Artifactum

Artifactum already provides the right data/computation vocabulary:

- content identity distinct from provenance;
- artifacts, collections, refs, leases, and graph-aware GC;
- ActionSpec/ActionKey, Attempt, and Realization;
- pure/reproducible/volatile/effect semantics;
- mapped per-item cache reuse;
- checkpoints, cancellation, logs, remote materialization, and verification.

The lesson is not to build a second pipeline cache inside Flow. Flow should compile function nodes into Artifactum actions and use actor publications as immutable graph inputs.

References: [Artifactum architecture](https://github.com/akiselev/artifactum/blob/master/docs/ARCHITECTURE.md), [execution semantics](https://github.com/akiselev/artifactum/blob/master/docs/EXECUTION.md).

### Outboard

Outboard provides a strong portable implementation boundary:

- executable discovery and shadow diagnostics;
- semantic interface requirements and capabilities;
- one-shot and persistent workers;
- size-bounded length-prefixed JSON;
- progress, output, cancellation, ping, and shutdown;
- OS-string-safe argv;
- reusable management/doctor commands;
- conformance testing.

Its own architecture explicitly separates packaging, discovery, application semantics, and transport. This makes Outboard the correct layer to generalize for Remoc and structured function/service contracts.

References: [Outboard README](https://github.com/akiselev/outboard/blob/master/README.md), [Outboard architecture](https://github.com/akiselev/outboard/blob/master/ARCHITECTURE.md), [worker protocol](https://github.com/akiselev/outboard/blob/master/crates/outboard-protocol/src/lib.rs).

### Daemonkit

Daemonkit's deliberate refusal to define an application protocol is valuable. It owns authenticated local lifecycle, readiness, generations, endpoint publication, restart, cleanup, and repair. Remoc/JSON/actor protocols should run over the stream Daemonkit hands them.

References: [Daemonkit README](https://github.com/akiselev/daemonkit/blob/master/README.md), [usage guide](https://github.com/akiselev/daemonkit/blob/master/docs/USAGE.md).

### Switchyard

Switchyard proves several service/code-generation ideas:

- preserve the original trait;
- generate typed backend-neutral clients and dispatchers;
- distinguish trusted native and isolated process backends;
- publish generations and pin calls;
- reject nonportable signatures at compile time;
- treat build identity and adversarial testing seriously.

The current macro's stringified Rust type schemas are not sufficient for agents/cross-language contracts. Facet provides the structural replacement.

References: [Switchyard README](https://github.com/akiselev/switchyard/blob/master/README.md), [public API](https://github.com/akiselev/switchyard/blob/master/docs/PUBLIC_API.md), [macro source](https://github.com/akiselev/switchyard/blob/master/crates/switchyard-macros/src/lib.rs).

## Facet and Figue

[Facet](https://github.com/facet-rs/facet) gives every reflected type a static `Shape` containing its structure, documentation, attributes, layout, and operations. [`facet-reflect`](https://docs.rs/facet-reflect/latest/facet_reflect/) can inspect and construct reflected values while preserving invariants. The ecosystem includes codecs, schemas, diffs, pretty-printing, and other integrations.

[Figue](https://docs.rs/figue/latest/figue/) demonstrates a compelling downstream use: one Facet type can drive CLI arguments, environment/config layers, defaults, help, completions, and schema output.

Lessons adopted:

- use one structural reflection source rather than parallel Clap/Serde/Schemars models;
- let function signatures lower into reflected request records;
- use reflected construction for CLI/KDL/JSON;
- keep a portable projection boundary because Facet's process-local operations/layout are not a wire ABI;
- pin/test the integration because the ecosystem is actively evolving.

## Remoc

[Remoc remote trait calling](https://docs.rs/remoc/latest/remoc/rtc/) is unusually well aligned with the proposed service layer. It generates clients, server variants, and request receivers; supports concurrent calls, pipelining, cancellation, monitoring, tracing, remote channels, and remote objects; and uses named request fields with useful additive compatibility behavior.

The request-receiver form is the key actor insight: calls can be processed as messages by an actor runtime instead of Remoc directly borrowing actor state. Remoc clients can also be used locally, reinforcing the “same semantic call, different placement” model.

Remoc intentionally does not provide discovery, authentication, encryption, load balancing, or durable actor state. These omissions match the intended composition:

- Outboard: discovery/capabilities/process placement;
- Daemonkit or network layer: secure session establishment;
- actor runtime: lifecycle/state/mailbox;
- Remoc: typed Rust transport.

Lessons adopted:

- use Remoc as the preferred Rust transport;
- preserve named request records;
- expose cancellation and pipelining capability;
- map actor calls through request receivers;
- do not make Serde/Remoc-generated types the cross-language contract;
- provide structured JSON fallback.

## KDL

The [KDL v2 specification](https://kdl.dev/spec/) defines a node-oriented document language with arguments, properties, child nodes, comments, slash-dash disabling, and type annotations. The official Rust [`kdl` crate](https://docs.rs/kdl/latest/kdl/) preserves formatting/comments and supports document editing with source spans.

Lessons adopted:

- graph declarations should look like graph nodes rather than nested map serialization;
- type annotations are useful for duration/URL/path/artifact scalars;
- comment-preserving edits materially improve agent DevEx;
- KDL should remain declarative; programmatic graph generation belongs in another frontend;
- canonical IR identity must ignore concrete formatting.

## Spider

[Spider](https://docs.rs/spider/latest/spider/) provides a configurable asynchronous crawler that can stream pages to subscribers. Its feature set includes control for pause/start/shutdown and multiple HTTP/browser modes. Its `Website` is a live runtime value with substantial mutable configuration and execution state.

Lessons adopted:

- reflect a stable `WebsiteSpec`, then hydrate `Website`;
- stream page artifacts into an open publication session;
- use actor services for status/refresh/control;
- keep pure parsers downstream of raw page evidence;
- advertise compiled feature/capability support before hydration;
- separate item streaming from sealed snapshot publication.

## Salsa

[Salsa](https://salsa-rs.github.io/salsa/overview.html) tracks dependencies of memoized functions and uses a red-green algorithm to avoid propagating changes when outputs remain unchanged.

Lessons adopted:

- function-oriented developer experience;
- dependency invalidation based on semantic input/output change;
- unchanged upstream results should keep downstream work reusable.

Not adopted directly:

- in-process database/query model as the durable data plane;
- mutation restrictions or macro-generated tracked calls as the user workflow model;
- automatic dependency discovery through arbitrary function calls across process/language boundaries.

Flow makes dependencies explicit in IR and applies the red/green idea to Artifactum identities and keyed collections.

## Buck2

[Buck2's query model](https://buck2.build/docs/concepts/buck_query_language/) distinguishes unconfigured, configured, and actual action graphs. Its [dynamic dependency documentation](https://buck2.build/docs/about/why/) shows both the utility and complexity of allowing data to influence later dependencies. [BXL](https://buck2.build/docs/bxl/) provides a programmable/introspective layer rather than bloating declarative target files.

Lessons adopted:

- expose declared graph, current plan, and actual trace separately;
- make query/explain first-class;
- restrict dynamic graph behavior so static analysis remains useful;
- use a separate programmable frontend/extension layer rather than adding expressions to KDL.

## Dagster

[Dagster assets](https://docs.dagster.io/api/dagster/assets) center persistent outputs and their producing functions rather than treating job execution as the product. Dagster also demonstrates structured external event integration through Dagster Pipes.

Lessons adopted:

- targets/publications/artifacts are primary user concepts;
- run history is operational evidence, not the result itself;
- external processes should stream structured events rather than only logs;
- data checks and lineage belong near artifacts.

Not adopted directly:

- Python definitions as the only graph authority;
- application-specific IO manager/resource abstractions where Artifactum and hydration already cover the need.

## Dagger

[Dagger's type-system guidance](https://docs.dagger.io/extending/type-system/) explicitly treats types as the contract used by CLI clients, SDKs, modules, agents, and compositional workflows. Its modules expose normal typed functions and rich object/service values, with generated CLI discovery and cross-language calls.

Lessons adopted:

- a component should be a typed API, not a named shell script;
- return rich values/artifacts instead of paths or log strings;
- function schemas should generate CLI/agent surfaces;
- services and secrets must be explicit types;
- interactive chaining/debugging should use the same function graph.

Difference:

- Flow separates persistent actors, Artifactum-backed durable artifacts, and an explicit declarative IR rather than making one object graph the entire execution model.

## Actor frameworks

[Ractor](https://docs.rs/ractor/latest/ractor/) demonstrates lifecycle, supervision, actor references, and failure propagation. Kameo and other actor crates explore actor refs and remote placement.

Lessons adopted:

- actor identity and supervision are first-class;
- mutable state should have an explicit serialization/mailbox domain;
- startup/failure/stop events need structured handling;
- references should not be raw process/socket handles.

Not adopted directly:

- a second independent remote actor protocol;
- transparent cluster semantics before local/process hosting is stable;
- modeling every finite Flow function as an actor message.

## Rust DAG/job/workflow crates

Crates such as Dagrs/Dagx provide graph nodes/dependencies; Apalis provides job-worker abstractions; durable workflow systems provide event-sourced replay. They are useful implementation precedents for scheduling or persistence but do not match the desired public model.

Principal conclusion:

- do not expose `add_node`/`add_edge` as normal application DevEx;
- do not reduce the system to a background job queue;
- do not require deterministic replay of arbitrary Rust workflow code;
- reuse Artifactum actions and explicit Flow IR instead.

## Agent CLI precedents

[Codex non-interactive mode](https://developers.openai.com/codex/non-interactive-mode) is explicitly separate from its interactive TUI and supports repeatable scripted operation. Codex developer commands support stdout/JSONL. [Claude Code headless mode](https://docs.anthropic.com/en/docs/claude-code/headless) supports text, JSON, and streaming JSON; its CLI supports stream JSON input and output.

Lessons adopted:

- every operation needs a one-shot non-interactive command;
- JSON/JSONL must be stable and complete;
- persistent REPL sessions should accept structured requests over stdin;
- no alternate-screen or prompt may be required;
- output must be bounded and provide artifact/continuation references.

## Research conclusion

The architecture is not “a Rust DAG crate with plugins.” It is the composition of:

```text
Facet      structural reflection and construction
Outboard   implementation discovery and placement
Remoc      preferred typed Rust service transport
JSON       language-neutral fallback
Daemonkit  persistent local lifecycle and authenticated streams
Artifactum immutable values, action identity, cache, lineage
Flow IR    explicit topology and incremental planning
Actors     persistent source/domain behavior and publications
KDL        editable declarative frontend
```

The design is differentiated by making all of these serve one discoverable function/service contract without requiring the user to understand the layers during ordinary authoring.
