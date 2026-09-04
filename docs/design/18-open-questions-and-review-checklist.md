# Open questions and review checklist

The package has a coherent baseline, but the following decisions should be explicitly reviewed before implementation design begins.

## 1. Final name and crate namespace

**Question:** Should the system be named `flow`, or should a less generic name avoid crate/command ambiguity?

**Recommended default:** Keep `Flow` as a working architectural name only. Select the public name after API vocabulary is approved.

Review:

- [ ] Command name does not collide unacceptably with common tools.
- [ ] Crate names can separate Outboard contract work from workflow-specific work.
- [ ] Names remain understandable in application CLIs (`auctions workflow ...`).

## 2. Function semantic annotations

**Question:** Use four macros (`derive`, `function`, `observe`, `effect`) or one macro with a `kind` attribute?

**Recommended default:** Four user-facing annotations sharing one implementation. They provide immediate semantic visibility and better defaults.

Review:

- [ ] `observe` is clearer than `source` for a finite external read.
- [ ] Actor publication remains the graph source concept.
- [ ] Pure/reproducible distinction remains explicit.

## 3. Contract derive spelling

**Question:** How should explicit semantic type identity attach to Facet types?

Options:

```rust
#[derive(Facet, Contract)]
#[contract(id = "...", version = 1)]
```

or a Facet custom attribute/attribute macro.

**Recommended default:** A narrow `Contract` derive beside `Facet`. It avoids rewriting the type and makes durable identity visible.

Review:

- [ ] Does not duplicate Facet reflection.
- [ ] Supports external/manual type adapters.
- [ ] Produces excellent missing-ID diagnostics.

## 4. Contract Shape scope

**Question:** How much validation/format metadata belongs in portable Contract Shape?

**Recommended default:** Include caller-visible structure, defaults, docs, formats, constraints, Flow semantic wrappers, and compatibility attributes; exclude UI-only presentation and process layout.

Review:

- [ ] Canonicalization is deterministic across builds/languages.
- [ ] Recursive/shared types are represented without unstable ordering.
- [ ] Fingerprints do not become compatibility policy accidentally.

## 5. Dynamic linked value carrier

**Question:** Use Facet `HeapValue`, `Box<dyn Any + Send>`, a generated vtable, or multiple fast paths?

**Recommended default:** Begin semantically with an opaque `OwnedValue` interface. Prototype Facet-owned value materialization and an Any-backed generated adapter; choose by safety/benchmark, not public API.

Review:

- [ ] No wire serialization.
- [ ] Moves non-Clone arguments/results correctly.
- [ ] Preserves field-path diagnostics.
- [ ] Does not expose unsafe lifetime/layout assumptions.

## 6. Registration mechanism

**Question:** `inventory`, `linkme`, explicit generated tables, or combinations?

**Recommended default:** Publicly support explicit registration and `Registry::linked()`. Keep linker mechanism private and sort/conflict-check deterministically.

Review:

- [ ] Works in normal native binaries and tests.
- [ ] Has an escape hatch for unusual targets.
- [ ] Does not rely on unspecified registration iteration order.

## 7. Service method authoring

**Question:** Permit many typed arguments or require one request struct?

**Recommended default:** Permit both; always lower to a named request record. Lint/recommend explicit request types for APIs expected to evolve.

Review:

- [ ] Defaults and aliases have portable semantics.
- [ ] JSON/Remoc request shapes agree.
- [ ] Adding defaulted fields is compatible.

## 8. Portable service feature set

**Question:** How much of Remoc's remote channels/objects/associated types should the base service contract expose?

**Recommended default:** Portable semantic wrappers (`RemoteStream`, `Subscription`, `ServiceRef`, `ActorRef`) mapped by each transport. Raw Remoc types require an explicit Remoc-only capability.

Review:

- [ ] Baseline services remain implementable in Python/Node.
- [ ] Remoc does not become merely a byte codec with all strengths discarded.
- [ ] JSON streaming semantics are well-defined.

## 9. Actor concurrency

**Question:** Strict mailbox serialization, Remoc-style shared mutable dispatch, or configurable modes?

**Recommended default:** Serialized actor domain/mailbox. Explicit opt-in for snapshot-safe concurrent reads. Long I/O runs as admitted child operations that return events to the mailbox.

Review:

- [ ] Manual calls remain responsive during crawls.
- [ ] State transitions are deterministic/auditable.
- [ ] Cancellation and quiesce have defined ordering.
- [ ] Linked and Remoc calls preserve the same semantics.

## 10. Actor state storage

**Question:** Exact API for small control state versus bulk checkpoints?

**Recommended default:** Transactional small state in actor/control storage; bulk/recoverable state in Artifactum checkpoints. Daemonkit state remains lifecycle-only.

Review:

- [ ] State schema/version/migration is explicit.
- [ ] Crash between publication seal and state update converges safely.
- [ ] Checkpoint retention/GC roots are defined.

## 11. Open publication semantics

**Question:** May downstream work begin before collection seal?

**Recommended default:** Yes for independently keyed mapped work; aggregates/targets wait for seal. Named publication advances atomically only at seal.

Review:

- [ ] Aborted sessions cannot appear complete.
- [ ] Per-item work remains reusable.
- [ ] deletion/partial snapshot semantics are explicit.
- [ ] backpressure and item-event limits are defined.

## 12. Actor-to-Flow feedback loops

**Question:** Can an actor consume a Flow output that ultimately depends on its own publication?

**Recommended default:** Permit only explicit feedback edges with generation/delay semantics; reject immediate dependency cycles.

Review:

- [ ] Two-stage discovery/detail-fetch can be expressed.
- [ ] No infinite re-plan loop from unchanged republishing.
- [ ] generation barriers are inspectable.

## 13. KDL composition

**Question:** Profiles/includes/overlays syntax and semantics.

**Recommended default:** Typed named profiles and explicit ordered overlays; root-scoped local includes only. No interpolation or remote import.

Review:

- [ ] Effective config has deterministic provenance.
- [ ] Agent edits preserve comments.
- [ ] Locking covers included files.
- [ ] No surprising current-directory behavior.

## 14. CLI parser integration

**Question:** Stock Figue CLI only, parser-neutral command model, or first-class Figue + Clap adapters?

**Recommended default:** Parser-neutral Facet command model and dispatcher; stock Figue-based frontend; supported Clap mounting adapter for existing apps.

Review:

- [ ] Existing Auctions/Foundry CLI can embed without a rewrite.
- [ ] Dynamic function arguments can be generated at runtime.
- [ ] help/completion/JSON schema stay consistent.

## 15. Error contract

**Question:** Require typed error derives or allow arbitrary errors?

**Recommended default:** Accept any convertible error; typed `Diagnostic` derive yields rich portable detail; arbitrary errors become stable unstructured envelopes.

Review:

- [ ] retryability is typed, never string-matched.
- [ ] secret redaction is shape-driven.
- [ ] cross-language errors preserve code/category/help.

## 16. Linked code identity

**Question:** How is a linked function's implementation digest derived?

**Recommended default:** containing binary/library artifact digest + function descriptor fingerprint + build metadata/attestation. Avoid source-path-only identity.

Review:

- [ ] reproducible enough for Artifactum cache reuse.
- [ ] debug/dev builds are distinguishable.
- [ ] custom CLI recompilation invalidates relevant actions.

## 17. Remote service identity in pure actions

**Question:** When can a service call participate in a pure/reproducible function?

**Recommended default:** Only when the call binds to an immutable service/model/index snapshot token and trusted implementation identity. Otherwise the function is volatile/observe.

Review:

- [ ] mutable endpoint names cannot masquerade as deterministic inputs.
- [ ] model/provider version evidence is retained.

## 18. Transport fallback moment

**Question:** Can fallback happen after a call was admitted/sent?

**Recommended default:** No, unless operation idempotency and retry policy explicitly authorize another attempt. Fallback normally occurs during resolution/session establishment.

Review:

- [ ] effects cannot duplicate silently.
- [ ] failure reports show attempted transports.
- [ ] cancellation/timeout ambiguity is preserved.

## 19. JSON v2 versus existing protocol evolution

**Question:** New independent protocol or additive frames in current worker protocol?

**Recommended default:** Independently versioned structured protocol carried by the same framing/session machinery; coexist with v1 in manifest negotiation.

Review:

- [ ] old workers remain compatible.
- [ ] one host can support both.
- [ ] frame limits and OS-string behavior remain clear.

## 20. Starlark frontend

**Question:** Include in the initial product surface?

**Recommended default:** Reserve the frontend interface and IR APIs; do not make Starlark necessary for the architecture review. KDL and Rust IR builders must cover the real examples first.

Review:

- [ ] KDL is not stretched to solve repetition.
- [ ] frontend-generated IR remains inspectable and source-mapped.

## 21. Network actor discovery

**Question:** How are remote actors found and authenticated?

**Recommended default:** Out of baseline scope. Define placement/discovery traits and stable endpoint identity requirements without selecting mDNS/Tailscale/registry mechanisms here.

Review:

- [ ] local/process architecture does not preclude network placement.
- [ ] Remoc is not mistaken for discovery/security.

## 22. Working with direct in-memory pipelines

**Question:** Should Flow support non-durable transient values between linked nodes?

**Recommended default:** Not implicitly. Direct Rust callers can compose functions in memory; orchestrated Flow uses durable artifact boundaries when results participate in caching/lineage.

Review:

- [ ] performance requirements are measured before adding a transient graph mode.
- [ ] provenance loss cannot happen silently.

## Architectural review checklist

### Mental model

- [ ] Function, Actor, Service, Flow, Artifact, Publication, Target, Run, Attempt, and Realization are distinct.
- [ ] A reviewer can explain when source behavior belongs in an actor versus an observe function.
- [ ] Nodes are bindings, not implementation classes.

### DevEx

- [ ] Minimal function/service/actor examples contain no scheduler plumbing.
- [ ] Direct ordinary Rust calls remain available.
- [ ] Generated help and diagnostics are credible for agents.
- [ ] A custom application CLI can avoid external worker overhead.

### Portability

- [ ] Semantic contracts do not depend on Remoc, Serde layout, or Rust type spelling.
- [ ] JSON fallback can implement baseline methods.
- [ ] Bulk data remains artifact-backed.

### Correctness

- [ ] Open/sealed publication semantics are unambiguous.
- [ ] Cache identity and remote implementation identity are honest.
- [ ] Effects/retries do not imply exactly-once.
- [ ] stale actor/workflow generations cannot commit incorrectly.

### Operations

- [ ] Declared graph, plan, and trace have separate APIs.
- [ ] Every TUI/REPL action has a structured one-shot equivalent.
- [ ] actor/service/function health and compatibility are discoverable.
- [ ] crash/replay/fork behavior is specified.

### Boundaries

- [ ] Artifactum, Outboard, Daemonkit, actor state, and control storage do not overlap.
- [ ] Existing internal repositories have a credible integration boundary.
- [ ] No hidden implementation plan has been mistaken for architecture.
