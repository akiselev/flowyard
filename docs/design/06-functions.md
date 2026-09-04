# Functions

## Authoring surface

A function is authored as ordinary Rust with one of four semantic annotations.

```rust
#[flow::derive(
    id = "auction.classify",
    version = 1,
    cache = "pure"
)]
async fn classify(
    context: &flow::Context,
    listing: Artifact<Listing>,
    #[flow(default = 0.72)] threshold: f32,
) -> Result<Classification, ClassifyError> {
    // domain implementation
}
```

The annotation MUST preserve `classify` as a normal function. It remains directly callable in unit tests and custom code.

## Generated argument record

The macro lowers the function's named parameters into a hidden reflected record:

```rust
#[derive(Facet)]
struct Args {
    listing: Artifact<Listing>,
    threshold: f32,
}
```

This named record is the basis for:

- KDL bindings;
- CLI flags and help;
- JSON requests;
- schema compatibility;
- defaults;
- generated documentation;
- linked type-erased invocation;
- replay/fork overrides.

Positional wire compatibility is not inferred from Rust parameter order.

## Function descriptor

Illustrative descriptor:

```rust
pub struct FunctionDescriptor {
    pub id: FunctionId,
    pub version: Version,
    pub docs: Documentation,
    pub kind: FunctionKind,
    pub cache: CacheSemantics,
    pub args: &'static Shape,
    pub result: &'static Shape,
    pub contract: fn() -> &'static FunctionContract,
    pub capabilities: CapabilityRequirementSet,
    pub execution: ExecutionHints,
    pub local: Option<&'static dyn LocalFunctionInvoker>,
}
```

`FunctionContract` is portable and contains Contract Shapes rather than process-local `Shape` pointers.

## Function kinds

### `derive`

Requirements:

- inputs affecting output are explicit and immutable;
- output is safe to treat under declared `pure` or `reproducible` semantics;
- external effects are prohibited;
- network access is normally denied unless reproducibility policy explicitly includes it, which should be exceptional.

The generated descriptor maps directly to Artifactum action/caching semantics.

### `function`

Default behavior:

- volatile;
- callable interactively;
- may return inline or artifact-backed values;
- no reuse is assumed unless the caller wraps it in an explicit action policy.

Useful for search, query, exploratory analysis, diagnostics, and administrative operations.

### `observe`

Requirements:

- reads mutable external reality;
- records observation time and source identity;
- emits immutable data plus an observation receipt;
- never pretends that a previous response is a current fetch;
- can use checkpoints and idempotent item identities.

An observe function is finite. A persistent actor invokes it according to source-specific policy.

### `effect`

Requirements:

- executes on every admitted call;
- requires an explicit effect capability/policy;
- emits an immutable receipt even when no data result exists;
- should accept or derive an idempotency key where the external API supports one;
- replay tooling warns and defaults to dry-run/inspection rather than re-execution.

## Inputs and outputs

### Value arguments

Small structured values are passed directly in linked mode and encoded in remote modes.

### Artifact arguments

`Artifact<T>` represents an immutable semantic artifact. Interactive syntax may accept:

```text
@latest-report
artifact:sha256:...
./local-file.pdf
```

A local path is imported first under an explicit `T` contract. The function receives an artifact handle, not an untracked path.

### Collection arguments

`Collection<T>` is a keyed collection artifact. Mapping is expressed by Flow IR, not hidden inside the function unless the function's algorithm genuinely consumes the whole collection.

### Actor/service references

`ActorRef<S>` or `ServiceRef<S>` is a stable reference resolved by the context. It is not a raw Remoc client in the semantic signature.

### Secrets

`Secret<T>` is a reference and capability request. Plain secret literals are rejected by default in persisted KDL/IR and redacted from events.

### Results

A function may return:

- a small reflected value;
- `Artifact<T>`;
- `Collection<T>`;
- a record containing multiple named outputs;
- an observation/effect receipt;
- a portable stream/subscription handle for interactive/service cases.

When a Flow node requires durable reuse, plain reflected values are committed through the declared artifact codec before realization. A direct ordinary Rust call outside Flow may use the value entirely in memory.

## Linked invocation paths

### Direct static call

```rust
let result = classify(&context, listing, 0.72).await?;
```

No registry, allocation, reflection, or serialization is required beyond what the function itself uses.

### Dynamic linked call

```text
Facet-guided Args construction
    -> owned reflected/type-erased Args
    -> generated linked invoker
    -> downcast/materialize Args
    -> ordinary function call
    -> reflected/type-erased result
```

This path has dynamic dispatch and value-erasure overhead but no JSON/Postcard/process serialization. The exact owned carrier—Facet `HeapValue`, `Box<dyn Any>`, or a generated vtable—is an implementation choice left open for benchmarking.

### External call

The runtime uses the portable function descriptor and selects Remoc, structured JSON, or legacy argv execution. Bulk inputs remain artifact references/materialized files.

## Function resolution

A node or interactive call specifies a semantic requirement:

```text
auction.classify @ ^1
```

Resolution considers:

- semantic ID/version;
- Contract Shape fingerprint/compatibility;
- function kind and cache semantics;
- required capabilities;
- platform/resource constraints;
- trust/isolation policy;
- available placement and transport;
- explicit implementation preference.

The chosen implementation and all rejected candidates are visible through introspection.

## Calling from the CLI

```text
flow function call auction.classify \
  --listing @lot-123 \
  --threshold 0.8
```

Agent-friendly alternative:

```text
flow function call auction.classify \
  --args-json '{"listing":"@lot-123","threshold":0.8}' \
  --format jsonl
```

`flow function describe auction.classify --format json` returns the descriptor, argument Contract Shape, result Contract Shape, implementations, placement options, policy needs, and examples.

## Calling from Flow IR

```kdl
map "classify" use="auction.classify@^1" {
    each "listing" from="parse.listings"
    arg "threshold" 0.72
}
```

The KDL frontend does not need hard-coded knowledge of `threshold`. It validates and constructs it through the resolved descriptor.

## Diagnostics and errors

Function failures use a structured envelope:

```rust
pub struct DiagnosticError {
    pub code: ErrorCode,
    pub message: String,
    pub category: ErrorCategory,
    pub retry: RetryDisposition,
    pub path: Option<ValuePath>,
    pub details: Option<ContractValue>,
    pub help: Vec<String>,
    pub causes: Vec<DiagnosticCause>,
}
```

Domain error types may derive/implement a conversion into this envelope. `anyhow`-style errors remain usable but are classified as unstructured and lose portable field-level details.

Retryability is not inferred from text matching. Functions or adapters provide a typed disposition such as:

```text
never
transient-after(duration)
rate-limited-until(time)
requires-operator
requires-new-input
```

The workflow policy may narrow but not silently broaden retryability.

## Events

Every dynamic invocation can emit:

```text
started
progress
diagnostic
log
artifact_published
item_published
checkpoint
finished
failed
```

Events are advisory/observational except terminal result and explicitly committed artifact/publication events. Human rendering consumes the same event stream as JSONL rendering.

## Cancellation

Cancellation semantics are declared per implementation:

- linked async function: cooperative token and future cancellation;
- Remoc call: remote cancellation at cooperative await points, with method opt-out where necessary;
- JSON worker: cooperative cancel frame, then process termination if policy requires;
- one-shot process: process-tree termination;
- external effect: cancellation may be indeterminate and must be reflected in receipt/error state.

A function descriptor advertises cancellation support; the caller must not assume it.

## Function composition

Functions compose through typed values and artifacts, not shell text. Direct Rust composition remains ordinary code. Declarative composition uses Flow IR bindings.

A function should not receive a generic `NodeContext` that lets it mutate the graph. Dynamic graph generation, if introduced, belongs to an explicit planner-function contract.

## Anti-patterns

The design rejects these as primary APIs:

```rust
workflow.add_node("parse", Box::new(...));
workflow.add_edge("crawl", "parse");
```

```yaml
command: "python parse.py {input} {output}"
```

```rust
async fn parse(input: PathBuf) -> PathBuf;
```

They expose scheduler representation, lose semantic typing, or make provenance dependent on untracked paths and strings.
