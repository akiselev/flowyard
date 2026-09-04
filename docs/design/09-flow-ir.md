# Flow IR

## Purpose

Flow IR is the sole canonical representation of project topology. KDL, Rust builders, imported JSON, and a future Starlark frontend all compile into it.

Flow IR is designed to be:

- versioned;
- deterministic;
- serializable;
- source-language independent;
- statically inspectable;
- typed through resolved Contract Shapes;
- suitable for graph diffs and signatures;
- separate from execution history.

It is not a bytecode for algorithms and not a persisted scheduler queue.

## Top-level model

Illustrative Rust model:

```rust
pub struct ProjectIr {
    pub ir_version: u32,
    pub project: ProjectMetadata,
    pub actors: BTreeMap<ActorName, ActorInstanceSpec>,
    pub flows: BTreeMap<FlowName, FlowDefinition>,
    pub profiles: BTreeMap<ProfileName, ContractValue>,
    pub policies: BTreeMap<PolicyName, PolicySpec>,
}

pub struct CompiledProject {
    pub ir: ProjectIr,
    pub source_map: SourceMap,
    pub canonical_digest: Digest,
}
```

`SourceMap` is deliberately excluded from the canonical digest. It preserves file, span, comments/annotation links, and frontend provenance for diagnostics and editing.

## Actor instance specification

```rust
pub struct ActorInstanceSpec {
    pub actor: ActorRequirement,
    pub config: ContractValue,
    pub placement: PlacementPolicy,
    pub lifecycle: LifecyclePolicy,
    pub publications: BTreeMap<PublicationName, PublicationExpectation>,
    pub policy: PolicyReferenceSet,
}
```

The descriptor resolved from `ActorRequirement` defines the actual config shape, services, and publications. The IR does not repeat generated schemas.

## Flow definition

```rust
pub struct FlowDefinition {
    pub docs: Option<String>,
    pub sources: BTreeMap<SourceName, SourceBinding>,
    pub nodes: BTreeMap<NodeName, NodeSpec>,
    pub targets: BTreeMap<TargetName, TargetSpec>,
    pub policy: PolicyReferenceSet,
}
```

## Source bindings

A source binding imports an immutable value into the graph:

```rust
pub enum SourceBinding {
    ActorPublication {
        actor: ActorName,
        publication: PublicationName,
    },
    ArtifactRef {
        reference: ArtifactReference,
        expected: ContractTypeRequirement,
    },
    Parameter {
        name: ParameterName,
        expected: ContractTypeRequirement,
    },
}
```

A source binding is not executable by itself. Actors or operators advance the underlying artifact/publication.

## Node operations

The recommended initial algebra is deliberately small.

### Call

Invoke one function once with bindings.

```rust
NodeOperation::Call
```

### Map

Invoke one function for each member of a keyed collection. One item produces one independently keyed action and result.

```rust
NodeOperation::Map {
    each: ArgumentName,
    over: Binding,
}
```

### FlatMap

Invoke one function per member where each result is itself zero or more keyed members. The runtime constructs a deterministic flattened collection with explicit key composition/collision policy.

### Collect

Construct one collection from multiple compatible collections or outputs. Collect is structural and should not hide domain aggregation logic.

### Join

Join keyed collections under an explicit join mode and key contract. Join creates tuples/records of artifact references; it does not implement arbitrary SQL.

```rust
pub enum JoinMode {
    Inner,
    Left,
    Full,
}
```

### Target

Targets are not nodes; they name node/source outputs and publication behavior.

## Why filter is not initially a graph primitive

Filtering often carries domain reasons, confidence, and audit evidence. A normal function returning a typed decision or `Option<T>` is more inspectable than a graph expression:

```rust
#[derive(Facet, Contract)]
struct PromotionDecision {
    accepted: bool,
    reasons: Vec<PromotionReason>,
}
```

A future optimized `select` operator may be justified, but the initial IR should not encode an expression language.

## Node specification

```rust
pub struct NodeSpec {
    pub operation: NodeOperation,
    pub function: FunctionRequirement,
    pub bindings: BTreeMap<ArgumentName, Binding>,
    pub outputs: BTreeMap<OutputName, OutputPolicy>,
    pub execution: ExecutionPolicyOverride,
    pub placement: PlacementPolicy,
    pub policy: PolicyReferenceSet,
}
```

## Bindings

```rust
pub enum Binding {
    Literal(ContractValue),
    Source(SourceOutputRef),
    Node(NodeOutputRef),
    MapItem,
    MapKey,
    Profile(ProfileReference),
    Secret(SecretReference),
    Actor(ActorReference),
}
```

Bindings contain no arbitrary expressions. A literal is validated against the resolved function argument Contract Shape.

## Output model

A function result is projected into named output ports. A scalar/single result has a conventional `result` port. A reflected record may expose declared output fields:

```rust
#[derive(Facet, Contract)]
#[contract(id = "auction.parse-result", version = 1)]
#[flow(outputs)]
struct ParseResult {
    listings: Collection<Listing>,
    receipt: Artifact<ParseReceipt>,
}
```

Output projection is part of the function descriptor, not guessed by the KDL parser.

## Function requirements

```rust
pub struct FunctionRequirement {
    pub id: FunctionId,
    pub version: VersionReq,
    pub contract_fingerprint: Option<Digest>,
    pub implementation: ImplementationSelector,
    pub required_capabilities: CapabilityRequirementSet,
}
```

Normal source files specify ID + semantic version requirement. Locking may record exact implementation/build and contract fingerprints separately.

## Placement policy

Placement should be declarative and capability-oriented:

```rust
pub struct PlacementPolicy {
    pub locality: LocalityPreference,
    pub isolation: IsolationRequirement,
    pub trust: TrustRequirement,
    pub transports: Vec<TransportPreference>,
    pub platform: PlatformConstraint,
    pub resources: ResourceRequirement,
}
```

The IR should not normally name a PID, socket, temporary path, or concrete scheduler worker.

## Validation pipeline

### 1. Parse/frontend validation

KDL/Rust/Starlark constructs a structurally valid `ProjectIr` with source spans.

### 2. Identity validation

Names, semantic IDs, version requirements, references, and duplicate definitions are checked.

### 3. Graph validation

References resolve, outputs exist, cycles are diagnosed, map/flat-map scopes are valid, and targets are reachable.

### 4. Contract validation

Resolved function/actor descriptors validate:

- argument names;
- literal values;
- artifact/collection element types;
- output types;
- actor publication contracts;
- service references;
- defaults and required fields.

### 5. Capability and policy validation

Required network, secret, filesystem, process, effect, and transport capabilities are checked before planning.

### 6. Placement feasibility

At least one compatible implementation/transport/placement exists, unless deferred resolution is explicitly allowed.

## Diagnostics

A useful error identifies both sides of the mismatch:

```text
Flow.kdl:31:9: type mismatch in node `classify`

  argument: auction.classify@1::listing
  expects:  Artifact<auction.listing@2>
  receives: Artifact<web.html-page@1>
  from:     parse.pages

31 |     input "listing" from="parse.pages"
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

help: use `parse.listings`, or insert a function that produces auction.listing@2
```

Machine diagnostics preserve codes, spans, contract IDs, and suggested edits.

## Canonicalization and identity

Canonical Flow IR identity includes:

- IR schema version;
- semantic actors/flows/nodes/targets;
- canonical literal values;
- version requirements and policies;
- graph-relevant ordering where meaningful.

It excludes:

- source filename and spans;
- comments and formatting;
- map insertion order where semantics are unordered;
- resolved implementation/build unless using a lock;
- runtime status and timestamps.

## Locks

A project lock is separate from Flow IR and may pin:

- external actor/plugin implementation builds;
- exact contract fingerprints;
- source profile versions;
- container/image digests;
- remote endpoint identities;
- provider resolutions.

It does not contain derived realizations; Artifactum owns those.

## Graph diffs

A semantic diff should report:

```text
actor config changed
function version requirement changed
literal argument changed
binding rewired
node added/removed
policy tightened/relaxed
target moved
implementation lock changed
```

Formatting-only KDL changes produce no semantic diff.

## Dynamic graph generation

Arbitrary runtime graph mutation is excluded from the baseline IR. Repetition and project generation may be expressed by:

- Rust builder frontend;
- optional Starlark frontend;
- generated/imported IR fragments.

A future planner function may emit a bounded, typed subgraph after examining an artifact, but it would be an explicit operation with its own contract, identity, and introspection—not ordinary control flow hidden in a transform.

## Queries

The IR should support stable queries used by CLI and agents:

- functions/services/actors used by a target;
- upstream/downstream nodes;
- transitive artifact type requirements;
- effectful nodes;
- network/secret requirements;
- possible placements;
- actor publications consumed;
- semantic diff between definitions.

These operate on the declared graph. Separate query surfaces operate on the current plan and actual trace.
