# Decision register

This register records the current recommended decisions. `Accepted for review` means the design package treats the decision as its baseline; it is not an implementation commitment.

| ID | Decision | Status | Primary reason |
|---|---|---|---|
| D-001 | Use Facet as the Rust reflection system. | Accepted for review | One reflected shape can support construction, CLI/config, documentation, schemas, diagnostics, and introspection. |
| D-002 | Project Facet `Shape` into a deterministic Contract Shape for durable and remote contracts. | Accepted for review | Facet's process-local layout and operations are not a stable cross-language ABI. |
| D-003 | Use Function, Actor, Service, and Flow as the four principal concepts. | Accepted for review | Separates finite computation, persistence, remote interface, and graph topology. |
| D-004 | Define a node as a function plus bindings and execution policy. | Accepted for review | Avoids exposing scheduler graph objects as the application API. |
| D-005 | Use `derive`, `function`, `observe`, and `effect` as function semantics. | Recommended | Distinguishes caching and external-world behavior without making sources a second graph language. |
| D-006 | Make actors the normal owner of refresh/backoff/staleness decisions. | Accepted for review | Source-specific policy is stateful and awkward in a generic DAG scheduler. |
| D-007 | Make actor publications immutable Artifactum artifacts or sealed keyed collection snapshots. | Accepted for review | Preserves snapshot correctness, lineage, and incremental reuse. |
| D-008 | Use Flow IR as the sole canonical graph representation. | Accepted for review | Multiple frontends can coexist without multiple execution semantics. |
| D-009 | Use KDL v2 as the default editable frontend. | Accepted for review | Node-oriented syntax, comments, type annotations, source spans, and formatting-preserving edits fit the graph. |
| D-010 | Keep KDL intentionally non-programmable. | Accepted for review | Prevents configuration from becoming a weak programming language. |
| D-011 | Permit Rust and future Starlark frontends only as Flow IR producers. | Accepted for review | Programmatic generation remains possible without privileged runtime behavior. |
| D-012 | Co-design Flow functions/services/actors with an Outboard transport fabric. | Accepted for review | Discovery, process placement, and remote invocation need one semantic descriptor model. |
| D-013 | Use Remoc as the preferred Rust-to-Rust persistent transport. | Accepted for review | Its remote traits, cancellation, pipelining, remote objects, and request receiver align with services and actor mailboxes. |
| D-014 | Retain structured Outboard JSON as the language-neutral fallback. | Accepted for review | Python, Node, JVM, and shell implementations must remain first-class. |
| D-015 | Keep Remoc-specific trait bounds out of the portable semantic contract. | Accepted for review | Rust transport constraints must not define cross-language compatibility. |
| D-016 | Keep large data out of RPC frames; pass artifacts, files, streams, and references. | Accepted for review | Preserves efficiency and builds on Artifactum rather than duplicating blob transfer. |
| D-017 | Make transport selection policy-driven rather than globally hard-coded. | Accepted for review | Trust, isolation, locality, compatibility, and platform can override performance preference. |
| D-018 | Make the stock CLI a thin application of an embeddable CLI/runtime crate. | Accepted for review | Projects can link hot Rust implementations and configure command placement. |
| D-019 | Make the REPL line-oriented with plain and JSONL modes. | Accepted for review | Works in terminals, pipes, Codex, Claude Code, CI, and test harnesses. |
| D-020 | Treat any richer TUI as an optional inline renderer. | Accepted for review | No alternate-screen or terminal interactivity may be required for operation. |
| D-021 | Use Spider as the canonical end-to-end example. | Accepted for review | It exercises live configuration, hydration, actor lifecycle, streaming, snapshots, and mapped derivation. |
| D-022 | Preserve Artifactum's Action/Attempt/Realization and cache model. | Accepted for review | The required identity, provenance, replay, and cache semantics already exist. |
| D-023 | Keep workflow control state separate from Artifactum metadata. | Accepted for review | Operational scheduling state and immutable computational history have different mutation models. |
| D-024 | Keep Daemonkit limited to daemon lifecycle and authenticated endpoints. | Accepted for review | Actor/service protocols and domain state should not leak into a lifecycle library. |
| D-025 | Preserve original functions, traits, structs, and impls in macro expansion. | Accepted for review | Direct calls and ordinary Rust testing must remain possible. |
| D-026 | Treat macro diagnostics and generated schemas as public API. | Accepted for review | Human and agent DevEx depends on precise errors and stable introspection. |
| D-027 | Require explicit semantic IDs for durable/remote types, functions, services, and actors. | Accepted for review | Rust paths and compiler type identities are not stable product contracts. |
| D-028 | Let schema fingerprints detect structural change, but let explicit versions govern compatibility. | Accepted for review | A hash is evidence, not a compatibility policy. |
| D-029 | Allow speculative per-item work from an open publication, but advance named snapshots only on seal. | Recommended | Combines low latency with atomic snapshot semantics. |
| D-030 | Do not define exactly-once execution for external effects. | Accepted for review | Receipts and idempotency keys are honest; exactly-once claims are not. |

## Decisions intentionally left open

The following are not settled by this package:

- final crate/project name;
- whether linked registration uses `inventory`, `linkme`, generated explicit tables, or more than one option;
- the exact owned type-erasure carrier for dynamic linked calls;
- the exact actor state-store API and concurrency opt-ins;
- KDL include/profile composition syntax;
- whether the stock parser is directly Figue-based or uses a parser-neutral command model with Figue and Clap adapters;
- how much Remoc-native functionality is exposed through portable stream/object abstractions;
- network discovery and identity beyond local/process-hosted operation.

Each has a recommended default in [Open questions and review checklist](18-open-questions-and-review-checklist.md).
