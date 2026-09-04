# Facet contracts and hydration

## Why Facet

Facet gives Rust types a `SHAPE` associated constant containing structural and operational reflection: kind, fields, variants, documentation, attributes, layout, and type-specific operations. `facet-reflect` can inspect existing values and safely construct values of arbitrary reflected shapes while preserving invariants.

That makes Facet suitable as the single Rust-side substrate for:

- function argument and result discovery;
- service method requests/results;
- actor configuration and state descriptions;
- KDL decoding and validation;
- CLI argument construction;
- JSON and other codecs;
- generated documentation and completion;
- interactive value inspection;
- structural diagnostics and diffs.

Figue demonstrates that Facet shapes can drive layered CLI, environment, config-file, defaults, help, completion, and schema behavior. The design should reuse that ecosystem rather than recreate a second field-reflection system.

## Facet Shape versus Contract Shape

Facet `Shape` is the in-process reflection authority. It is not itself the durable or cross-language contract.

A deterministic projection, called **Contract Shape**, carries the portable subset:

```rust
pub struct ContractShape {
    pub schema_version: u32,
    pub id: ContractTypeId,
    pub version: ContractVersion,
    pub docs: Option<String>,
    pub definition: ContractDef,
    pub attributes: BTreeMap<String, ContractValue>,
    pub fingerprint: Digest,
}
```

Illustrative definitions:

```rust
pub enum ContractDef {
    Unit,
    Bool,
    Integer(IntegerDef),
    Float(FloatDef),
    String(StringDef),
    Bytes,
    Option(Box<ContractRef>),
    List(Box<ContractRef>),
    Map { key: ContractRef, value: ContractRef },
    Tuple(Vec<ContractRef>),
    Struct(Vec<ContractField>),
    Enum(Vec<ContractVariant>),
    Semantic(SemanticDef),
}
```

Contract Shape includes:

- serialized field and variant names;
- field order where semantically relevant;
- required/optional/default status;
- numeric ranges and string formats when declared;
- nested semantic type references;
- documentation intended for callers;
- Flow-specific classifications such as artifact, collection, secret, duration, URL, actor reference, and stream;
- compatibility-relevant attributes.

It excludes:

- size and alignment;
- field offsets;
- drop/clone/default vtables;
- pointer addresses;
- Rust `TypeId`/Facet `ConstTypeId` as external identity;
- crate/module source paths unless explicitly documentation-only;
- compiler version;
- private fields not exposed by the contract;
- non-semantic formatter/editor state.

## Explicit semantic type identity

Any type used for durable artifacts, Flow ports, service boundaries, or remote actor configuration MUST have an explicit semantic ID and version.

Proposed authoring form:

```rust
#[derive(Facet, Contract)]
#[contract(id = "auction.listing", version = 2)]
pub struct Listing {
    pub source: SourceId,
    pub external_id: String,
    pub title: String,
    pub price: Option<Money>,
}
```

`Contract` does not duplicate reflection. It supplies semantic identity, validates the Facet shape, and exposes the portable projection.

Plain internal/config types may derive only `Facet` when they never cross a durable or remote boundary. A macro that exports them will emit a targeted error if a semantic contract is required.

## Contract fingerprints

The fingerprint is a digest of canonical Contract Shape bytes. It is used to:

- detect implementation disagreements;
- cache generated schemas/help;
- explain graph-validation changes;
- compare plugin manifests;
- reject accidental same-version structural drift when configured strictly.

The fingerprint is not the compatibility authority. Explicit semantic version requirements remain authoritative because structurally different schemas can be intentionally compatible, and structurally identical schemas can have different semantics.

## Construction and dynamic values

CLI, KDL, JSON, and REPL inputs should all lower through the same construction pipeline:

```text
text/KDL/JSON token
    |
Contract Shape-guided parse
    |
Facet Partial / reflected builder
    |
fully initialized Rust Args value
    |
validation
    |
linked call or wire codec
```

No independent `FromStr` table should be built for every command parser.

The default scalar behavior is conventional:

| Rust/contract shape | CLI/KDL behavior |
|---|---|
| `String`, integer, float | one scalar value |
| `bool` | explicit boolean or positive/negative flag |
| `Option<T>` | omitted or one value |
| `Vec<T>` | repeated value or list syntax |
| unit enum | closed choice |
| data enum | tagged choice with nested fields |
| struct | named fields, optionally nested/flattened by declared policy |
| `PathBuf` | platform path; never forced through UTF-8 in process argv mode |
| `Url` | validated URL semantic type |
| `Duration` | duration grammar such as `250ms`, `3h` |
| `Artifact<T>` | ref/digest/path import syntax constrained to `T` |
| `Collection<T>` | collection artifact reference |
| `Secret<T>` | secret reference, never literal by default |
| `ActorRef<S>` | actor instance/service reference |

Custom scalar parsing should use a Facet-visible parsing capability or semantic adapter rather than hidden parser-specific hooks.

## Codecs

Facet reflection enables multiple codecs, but codec choice is placement-specific:

- linked invocation: no serialization;
- structured JSON fallback: Facet JSON/contract-value encoding;
- Remoc: Remoc-supported Serde/Postcard-style encoding and channel machinery;
- Artifactum structured artifact: canonical declared codec, often canonical JSON, CBOR, Arrow, Parquet, or domain media;
- CLI argv: OS-string-safe parser/formatter;
- KDL: Contract Shape-guided KDL decoder.

A type's semantic identity does not imply one universal byte encoding. Artifact manifests and method/function descriptors declare the codec/media used at a particular boundary.

## Hydration

### Problem

Many valuable inputs are not serializable values:

```text
spider::Website
sqlx/Postgres pool
browser context
loaded embedding model
search index writer
authenticated API session
remote actor client
GPU runtime
```

Trying to derive Facet/Serde for these values would either fail or expose unstable implementation details.

### Proposed trait

```rust
pub trait Hydrate: Facet<'static> + Sized {
    type Runtime: Send + 'static;

    async fn hydrate(
        self,
        context: &HydrateContext,
    ) -> Result<Self::Runtime, HydrateError>;
}
```

The reflected type is the stable specification. The runtime type exists only in its placement.

`HydrateContext` provides explicitly authorized capabilities:

```rust
pub struct HydrateContext<'a> {
    pub artifacts: &'a ArtifactStore,
    pub secrets: &'a SecretResolver,
    pub services: &'a ServiceResolver,
    pub files: &'a FileResolver,
    pub network: &'a NetworkPolicy,
    pub processes: &'a ProcessPolicy,
    pub cancellation: CancellationToken,
    pub diagnostics: &'a DiagnosticSink,
}
```

Hydration MUST NOT read ambient secrets or arbitrary files simply because the process can. The spec and policy jointly determine available capability.

### Spider example

```rust
#[derive(Facet, Contract)]
#[contract(id = "spider.website-spec", version = 1)]
pub struct WebsiteSpec {
    pub seed: Url,
    pub limits: CrawlLimits,
    pub routing: UrlRouting,
    pub transport: HttpPolicy,
    pub browser: BrowserPolicy,
}

impl Hydrate for WebsiteSpec {
    type Runtime = spider::website::Website;

    async fn hydrate(
        self,
        context: &HydrateContext,
    ) -> Result<Self::Runtime, HydrateError> {
        // Build and validate the live Website here.
    }
}
```

`WebsiteSpec` is usable from KDL, CLI, actor configuration, tests, and manifests. `Website` remains a live crawler object.

## Resource injection versus hydration

Function arguments SHOULD be durable/portable values. Common runtime resources should normally be injected through `Context`:

```rust
async fn classify(
    context: &flow::Context,
    listing: Artifact<Listing>,
) -> Result<Classification, Error> {
    let model = context.service::<ClassifierService>("default").await?;
    // ...
}
```

Hydration is appropriate when the runtime resource is the explicit product of a user-visible spec, particularly actor creation. It should not turn every function invocation into a service locator.

## Configuration layering

Facet/Figue-style layering is useful, but canonical actor/function identity requires clarity. The effective configuration should be constructed from ordered layers:

```text
code defaults
< project profile
< environment-selected profile
< CLI/REPL override
< explicit per-call value
```

The runtime records the fully resolved canonical value and provenance of each layer. Ambient environment values are not silently omitted from action identity when they affect computation.

Secrets are recorded as stable secret references and, where policy permits, version/fingerprint metadata—not plaintext.

## Validation levels

1. **Syntactic:** value can be constructed from KDL/CLI/JSON.
2. **Structural:** required fields, variants, ranges, and custom shape constraints are valid.
3. **Semantic:** domain validation implemented by the type/spec.
4. **Capability:** hydration/invocation policy permits required resources.
5. **Runtime:** live resource can initialize and pass readiness/doctor checks.

Errors should preserve a field path across all five levels.

## Facet stability boundary

Facet is active and its ecosystem documents varying maturity. The design limits risk by:

- pinning compatible Facet releases at workspace boundaries;
- centralizing Facet-to-Contract-Shape projection;
- snapshot-testing portable schemas;
- never persisting raw Facet layout/vtable representation;
- exposing stable semantic descriptors from our crates;
- permitting internal adaptation to Facet API evolution without changing the external protocol.

Facet is foundational reflection, not the external ABI.
