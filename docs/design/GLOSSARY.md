# Glossary

**Action** — Artifactum's canonical computation request, identified by an ActionKey.

**Actor** — A persistently hosted component with identity, configuration, state, lifecycle, services, wakeups, and named publications.

**Actor generation** — One concrete hosted generation of an actor instance. References may pin or follow generations according to policy.

**Artifact** — An immutable semantically described value in Artifactum. Artifact identity is distinct from provenance.

**Attempt** — One execution of an action or admitted operation.

**Binding** — A connection from a function argument to a literal, source, node output, map item, secret, profile, or actor/service reference.

**Capability** — A declared resource/authority requirement or implementation feature, such as network access, a secret, browser support, effect permission, or a transport.

**Collection** — An immutable keyed mapping from logical item keys to artifact identities.

**Contract** — The explicit semantic identity/version and portable shape of a value, function, service, or actor boundary.

**Contract Shape** — Deterministic portable projection from Facet reflection used for manifests, compatibility, validation, schemas, and agents. It is not a Rust ABI.

**Control plane** — Mutable operational state for runs, readiness, leases, retries, priorities, admission, budgets, and actor lifecycle.

**Daemon instance** — A Daemonkit-owned local process/lifecycle identity, distinct from any actor instances hosted inside it.

**Declared graph** — The function/actor/source/target topology in Flow IR.

**Derive function** — A cacheable computation over immutable inputs, declared pure or reproducible.

**Effect function** — An externally visible mutation that always executes when admitted and emits a receipt.

**Flow** — A declarative graph binding typed functions to immutable sources and actor publications.

**Flow IR** — The canonical, versioned, source-language-independent representation of actors, flows, nodes, bindings, targets, and policies.

**Function** — A typed finite callable operation with semantic identity, arguments, result, execution semantics, implementations, and events.

**Hydrate** — Construction of a live runtime object from a reflected stable specification inside an authorized context.

**Implementation** — Concrete linked code, executable plugin, actor host, or remote endpoint satisfying a semantic function/service/actor requirement.

**KDL frontend** — The default human-editable syntax that compiles to Flow IR.

**Linked invocation** — In-process Rust invocation without wire serialization. It may be a direct static call or a dynamic registry call.

**Node** — A function requirement plus argument bindings and execution policy.

**Observe function** — A finite read of mutable external reality that emits immutable evidence and an observation receipt.

**Open publication session** — An unsealed actor publication under construction. Item artifacts may exist and trigger mapped work, but the named publication has not advanced.

**Plan** — Concrete cache/execute/block/placement decisions for making targets current at a particular source state.

**Publication** — A named typed actor output pointing to an immutable artifact or sealed collection snapshot.

**Realization** — A successful binding from an Artifactum action identity to named immutable outputs.

**Receipt** — Immutable evidence of an observation or external effect, including attempt and external identity where available.

**Remoc transport** — Preferred Rust-to-Rust service transport. It is an adapter beneath the semantic contract.

**Run** — One orchestration episode attempting to make one or more targets current.

**Sealed collection** — An immutable complete or explicitly classified snapshot that may advance a named publication.

**Service** — A typed persistent trait/interface implemented by an actor or another hosted component.

**Source binding** — A Flow IR root that imports an actor publication, artifact ref, or run parameter into a graph.

**Target** — A named desired Flow output.

**Trace** — The record of what actually executed, emitted, retried, failed, and realized.

**Transport** — Linked, Remoc, JSON, argv, or future placement-specific mechanism carrying calls/events after semantic resolution.

**Workflow control store** — Mutable persistence for scheduling/admission/leases/retries, separate from Artifactum's immutable computation/data model.
