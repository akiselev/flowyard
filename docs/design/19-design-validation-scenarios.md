# Design validation scenarios

These are architecture acceptance scenarios, not an implementation order. A proposed API or design change should be evaluated against all applicable scenarios.

## Scenario 1: direct ordinary Rust call

Given an annotated pure function:

```rust
#[flow::derive(id = "auction.parse", version = 1, cache = "pure")]
async fn parse(context: &Context, page: Artifact<HtmlPage>)
    -> Result<ParseResult, ParseError>;
```

A unit test calls `parse(&context, page).await` directly.

Acceptance:

- no registry required;
- no serialization;
- original signature and error type visible;
- macro-generated adapters do not affect normal testing.

## Scenario 2: dynamic linked function in custom CLI

An Auctions binary links parser/classifier crates, embeds `flow-cli`, loads the same KDL project, and runs the target.

Acceptance:

- registry selects linked implementations;
- dynamic invocation does not encode/decode JSON/Postcard between linked functions;
- Artifactum identities/provenance remain complete for durable nodes;
- `flow function describe` reports linked placement/build identity;
- external optional functions can coexist.

## Scenario 3: external Rust worker over Remoc

The stock `flow` CLI discovers an Outboard Rust plugin implementing the same function/service and negotiates Remoc.

Acceptance:

- semantic descriptor matches linked version;
- arguments/results behave identically;
- large inputs pass as artifact references;
- progress and cancellation work;
- worker persists across calls where configured;
- transport choice is visible.

## Scenario 4: non-Rust JSON fallback

A Python or Node plugin implements a baseline function/service through structured JSON v2.

Acceptance:

- no Rust-generated code is required to understand the contract;
- plugin manifest/descriptor can be validated before calls;
- required/default fields and errors match Rust behavior;
- events and cancellation are coherent;
- bulk outputs are staged/imported and host-verified;
- unsupported Remoc-only features fail during resolution, not mid-call.

## Scenario 5: same Flow IR across placements

The same KDL/Flow IR is executed with linked, Remoc, and JSON implementations selected by policy.

Acceptance:

- graph does not change merely because placement changes;
- action identity changes only where implementation/build is output-relevant;
- target/result types remain identical;
- compatibility report explains selection.

## Scenario 6: Facet-driven function call

A user invokes an unfamiliar function through CLI without application-specific command code.

Acceptance:

- `describe --format json` exposes argument/result Contract Shapes, docs, defaults, examples, and capabilities;
- CLI flags are generated correctly;
- `--args-json` constructs the same request;
- local file supplied for `Artifact<Pdf>` is imported and reported;
- invalid nested field produces a path/span-aware diagnostic.

## Scenario 7: KDL agent edit

A coding agent changes a threshold and adds a node using semantic edit commands or direct KDL modification.

Acceptance:

- comments/unrelated formatting survive;
- `flow check` resolves function descriptors and types;
- typo/type errors identify exact span and valid alternatives;
- semantic diff distinguishes config/binding changes from formatting;
- the agent can obtain all results as JSON.

## Scenario 8: Spider actor hydration

A `WebsiteSpec` from KDL hydrates into a live Spider `Website`.

Acceptance:

- unsupported compiled feature is detected before crawl;
- network/browser/secret policies are checked explicitly;
- runtime object is not serialized/persisted;
- effective spec and hydration diagnostics are introspectable;
- restart recreates the runtime from spec and durable state.

## Scenario 9: manual and autonomous refresh converge

A Spider actor is due for refresh while an operator also requests `refresh --force`.

Acceptance:

- actor serializes admission;
- one domain operation path handles both triggers;
- overlap policy is explicit;
- callers receive operation IDs/status rather than blocking mailbox indefinitely;
- refresh decision and suppression/coalescing are visible.

## Scenario 10: streamed page processing with sealed snapshot

A crawl emits 300 pages over time.

Acceptance:

- each page is committed as an immutable artifact;
- mapped parse/classify work may start immediately;
- aggregate rank target does not use the collection as complete until seal;
- final publication advances atomically;
- unchanged pages reuse previous and speculative realizations.

## Scenario 11: crawler crash mid-publication

The crawler process exits after 180 pages.

Acceptance:

- previous sealed `pages` publication remains current;
- open session is marked aborted/recoverable;
- page artifacts, logs, and checkpoints remain inspectable;
- completed per-page derivations can be reused;
- retry resumes or restarts according to source semantics;
- no partial complete snapshot is exposed.

## Scenario 12: source deletion semantics

A later complete crawl omits a previously present URL; a partial API response also omits it.

Acceptance:

- complete snapshot may remove the key;
- partial snapshot does not infer deletion;
- delta/tombstone mode is explicit;
- downstream joins/reconciliation receive correct change reason.

## Scenario 13: stale workflow attempt

A leased attempt loses its lease, another worker succeeds, then the stale worker returns success.

Acceptance:

- stale attempt cannot commit terminal workflow success or advance publication;
- any staged bytes remain untrusted/unreferenced unless independently imported by policy;
- operational trace shows both attempts and rejection reason.

## Scenario 14: actor generation replacement

An actor config or implementation changes while clients hold references.

Acceptance:

- pinned references fail with generation-changed/incompatible error;
- follow-current references re-resolve only under declared policy;
- old generation quiesces and cannot publish after replacement;
- state migration is transactional;
- client/trace identifies called generation.

## Scenario 15: service cancellation

A client drops/cancels a long service request.

Acceptance:

- linked, Remoc, and JSON implementations advertise actual cancellation guarantees;
- actor mailbox/child task receives cancellation coherently;
- effect calls can report indeterminate outcome;
- transport failure is distinct from domain cancellation.

## Scenario 16: transport establishment failure

Remoc negotiation fails before a function call begins; JSON fallback is available.

Acceptance:

- fallback is policy-permitted and recorded;
- no duplicate semantic call occurs;
- required capabilities remain satisfied;
- diagnostic includes candidates/transports and reasons.

## Scenario 17: failure after call admission

Connection fails after an effect request may have reached the implementation.

Acceptance:

- no automatic fallback/retry unless explicit idempotency policy permits it;
- result is typed as indeterminate when appropriate;
- receipt/reconciliation information is retained;
- UI does not report ordinary failure as proof that no effect occurred.

## Scenario 18: reproducible service dependency

A derive function calls a pricing/model service.

Acceptance:

- pure/reproducible semantics require a bound immutable service/model snapshot token;
- mutable endpoint alone forces volatile/observe classification or validation failure;
- implementation/snapshot identity enters action key and explain output.

## Scenario 19: headless Codex/Claude operation

An agent invokes one-shot commands and a persistent JSONL REPL through pipes.

Acceptance:

- no TTY, prompts, ANSI, or alternate screen required;
- request IDs correlate interleaved events;
- stdout remains valid JSON/JSONL;
- large results spill to artifacts/files with continuation references;
- every interactive action has a command schema.

## Scenario 20: bounded malformed peer

A plugin sends oversized frames, invalid JSON, wrong schema, duplicate terminal events, or floods logs.

Acceptance:

- frame/event/resource bounds terminate or isolate the peer;
- host remains responsive;
- no unverified output is committed;
- protocol violation is distinct from domain error;
- doctor/conformance tooling can reproduce the failure.

## Scenario 21: semantic version evolution

A service adds a method and a defaulted request field without changing major version.

Acceptance:

- old and new clients interoperate under declared rules;
- Contract Shape fingerprint difference is explained;
- JSON and Remoc adapters agree;
- same-version incompatible drift is rejected.

## Scenario 22: KDL formatting-only change

Comments and formatting change without semantic IR change.

Acceptance:

- Flow IR digest remains unchanged;
- no target becomes stale;
- source-map positions update;
- semantic diff reports no graph change.

## Scenario 23: Auctions two-stage acquisition

A discovery actor publishes stubs, local prefilter selects a subset, and a detail-fetch actor observes selected URLs.

Acceptance:

- feedback relationship is explicit;
- no uncontrolled dependency cycle;
- source/detail budgets and checkpoints are separate;
- unchanged stubs/details reuse prior actions;
- application-specific prioritization remains outside generic scheduler code.

## Scenario 24: PartFoundry evidence and policy

A source actor fetches manufacturer evidence, networkless parser emits Bronze artifacts, reconciliation emits Silver candidates, and promotion checks policy/authority.

Acceptance:

- exact profile/policy/plugin/input identities are committed;
- Artifactum owns bytes; control DB stores references/operations;
- parser runs without network when declared;
- rights/authority decisions remain application contracts;
- stale attempts cannot promote;
- generic Flow code does not reinterpret evidence as truth.

## Scenario 25: direct custom domain command

The Auctions CLI exposes `auctions scan` as a projection of an actor service and `auctions search` as a function call.

Acceptance:

- generated argument/schema behavior is reused;
- domain command does not shell out to `flow`;
- direct linked implementation is selected;
- operation remains visible in normal Flow run/actor logs and events;
- stock framework commands may still be mounted under `auctions workflow`.
