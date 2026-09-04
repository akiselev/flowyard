# Security, versioning, and compatibility

## Threat model

The system may execute:

- trusted linked Rust code;
- trusted or untrusted local executables;
- persistent worker processes;
- browser/crawler code with network access;
- plugins in other languages;
- remote services;
- effectful calls with credentials;
- agent-supplied KDL/arguments.

No single execution mode provides the same isolation or trust properties.

## Trust and isolation modes

### Linked

- arbitrary trusted machine code in the host process;
- no memory/fault isolation;
- fastest local invocation;
- access must still be capability-scoped by API, but malicious linked code can bypass policy.

### Process

- crash and address-space isolation;
- optional filesystem/network/user/container sandboxing;
- host verifies outputs before Artifactum commit;
- process may still be malicious within granted OS capability.

### Wasm/shared memory/native optional backends

These may be supplied by Switchyard/other adapters. Each has a distinct threat model and must not be described simply as “plugin.”

### Remote

- requires authenticated peer identity and encrypted transport where crossing trust boundaries;
- Remoc provides RPC, not authentication/encryption;
- remote build/deployment identity must be attested or calls treated as volatile/untrusted;
- artifact transfer requires digest verification.

## Capability model

Descriptors declare requirements such as:

```text
network: host/scheme/port policy
filesystem: read/write roots
secrets: named references and purpose
process: executable/container/browser
artifacts: read/write kinds and stores
effects: named external action
remote: endpoint/identity class
resources: CPU/memory/GPU/time/cost
```

Capabilities are evaluated at definition, hydration, admission, and execution. A plugin manifest advertising support does not grant permission.

## Secrets

Secrets are represented as references:

```text
secret:ebay-api
secret:ti-oauth
```

Requirements:

- plaintext never appears in KDL, canonical IR, descriptors, logs, help, completion, or ordinary receipts;
- hydration/invocation resolves only declared references;
- secret values are scoped to the implementation/operation requiring them;
- environment-variable injection is explicit and redacted;
- cache/action identity records a safe secret version/fingerprint only when output-relevant and policy allows;
- remote forwarding requires explicit policy;
- JSON debug dumps redact by shape, not key-name heuristics.

## Artifact trust

Plugins return staged bytes or artifact claims. The trusted host:

- computes/verifies content digests;
- validates declared output type/media/schema;
- commits atomically only after successful execution;
- records producer/attempt provenance separately;
- rejects path escape/symlink attacks in sandboxes;
- verifies remote materialization.

An implementation cannot assign trusted Artifactum identity by string assertion alone.

## KDL and IR safety

KDL parsing/evaluation has no code execution. Includes, if enabled, are root-scoped and networkless. Unknown fields are errors by default.

Agent edits should support policy gates:

- semantic diff shown before effect/capability expansion;
- no silent activation of disabled nodes;
- lockfile changes separated from formatting changes;
- effectful/network/secret additions highlighted;
- machine-applicable edits carry precondition hashes to avoid stale writes.

## Actor identity and authorization

Actor calls identify:

- actor type and instance key;
- host/tenant/isolation domain;
- generation policy;
- service/method requirement;
- caller identity/capabilities.

Actor references are capabilities, not merely names. A human-readable name may require resolution to an authenticated internal reference.

A stale generation call fails explicitly unless follow-current behavior was requested.

## Transport security

### Daemonkit local sessions

Daemonkit authenticates local application streams and protects lifecycle ownership. Remoc/JSON runs inside that established channel.

### Child stdio

Parent-owned pipes inherit process identity but do not sandbox the child. Handles and environment must be minimized.

### Network

A network placement layer must provide:

- peer authentication;
- transport encryption;
- replay/session protection;
- endpoint authorization;
- bounded message/frame sizes;
- connection/resource limits;
- audit identity.

Outboard contract negotiation occurs after secure channel establishment.

## Denial of service controls

- maximum frame/message size;
- maximum nesting/container sizes in reflected decoders;
- request concurrency and queue bounds;
- per-actor mailbox limits;
- progress/log rate limits;
- artifact byte/time budgets;
- cancellation/deadline;
- process memory/CPU constraints where supported;
- recursive Contract Shape depth limits;
- bounded diagnostics and output tails.

Remoc and JSON adapters must enforce equivalent semantic limits even when their wire mechanics differ.

## External effects

Effect calls require:

- explicit effect descriptor;
- policy/admission;
- immutable operation request digest;
- optional confirmation token bound to that digest;
- idempotency key when possible;
- receipt with external request/response identity;
- typed `indeterminate` result when crash/timeout leaves outcome unknown;
- reconciliation method where external API permits.

Automatic transport retry after call admission is prohibited unless idempotency/retry policy explicitly permits it.

## Version domains

The following versions are independent:

1. Flow framework/library version;
2. Flow IR schema version;
3. KDL frontend grammar/profile version;
4. Contract Shape schema version;
5. semantic type version;
6. function semantic version;
7. service semantic version;
8. actor type version;
9. actor state/checkpoint schema version;
10. artifact schema/media version;
11. Outboard manifest version;
12. Outboard JSON worker protocol version;
13. Remoc adapter/session protocol version;
14. concrete implementation/plugin version;
15. implementation build identity.

Conflating these domains would make upgrades either unsafe or unnecessarily breaking.

## Semantic versioning rules

### Contract types

A major version is normally required for:

- removing/renaming a required field;
- changing field semantic meaning;
- changing scalar/container kind incompatibly;
- removing enum variants clients may send;
- changing artifact semantic type.

Potentially compatible changes include:

- adding an optional/defaulted field;
- adding a result field ignored by older clients;
- adding enum variants only when unknown-variant behavior is defined;
- strengthening documentation without validation changes.

### Functions

Breaking changes include:

- argument/result contract break;
- semantic meaning or side-effect class change;
- pure/reproducible claim no longer valid;
- removed capability/behavior relied upon by callers;
- renamed argument without alias/migration.

Implementation performance changes do not change function semantic version, but implementation build identity changes action keys.

### Services

Compatible evolution may include:

- adding a method;
- adding a defaulted request field;
- adding optional result fields;
- adding a transport implementation.

Breaking changes include removing methods, renaming wire methods, changing receiver/lifecycle semantics materially, or incompatible request/result/error changes.

Remoc's named request argument behavior is useful precedent, but the portable compatibility checker remains authoritative.

### Actors

Actor type version covers config, declared services/publications, and lifecycle semantics. Actor state schema has its own version and migration path.

A new implementation build does not necessarily change actor semantic version. A running actor generation changes whenever hosted implementation/config/state generation changes.

## Schema fingerprints

Every descriptor may include Contract Shape fingerprints. Uses:

- detect same-version disagreement;
- cache generated CLI/schema docs;
- explain changes;
- enforce locked exact contracts;
- compare language implementations.

Fingerprints never replace semantic version requirements. A compatibility report should show both:

```text
service version: compatible (^1)
shape fingerprint: differs
reason: optional field `force` added with default false
verdict: compatible
```

## Compatibility negotiation

Resolution checks:

1. manifest/framework protocol;
2. semantic ID/version;
3. Contract Shape compatibility;
4. required method/function capabilities;
5. transport capabilities;
6. policy/trust/isolation;
7. implementation health;
8. lock constraints.

A chosen implementation is recorded with the full compatibility report.

## Unknown fields and variants

- Input request/config unknown fields are errors by default.
- Explicit extension maps allow forward data without silent typos.
- Output readers may ignore unknown additive fields when the contract marks them extensible.
- Enum unknown variants require explicit representation; do not silently map to the first/default variant.

## State migration

Actor state/checkpoint migration is explicit:

```text
old state schema + actor type/build
    -> named migration implementation
    -> new immutable checkpoint/control transaction
```

Failure leaves the old generation/state intact. Native hot-reload generation behavior should follow Switchyard's conservative lesson: publish a new generation; do not assume arbitrary state/code can be unloaded or mutated safely.

## Supply-chain identity

Implementation records should support:

- executable/container digest;
- package/version;
- build ID;
- source commit;
- signer/attestation;
- SBOM/provenance references;
- lockfile pin.

Policy may require signed/locked builds for effects or sensitive sources while permitting local development builds for pure fixture processing.

## Audit and redaction

Structured events and receipts include stable operation IDs and policy decisions. Redaction is driven by Contract Shape annotations and capability classes.

Audit records distinguish:

- requested operation;
- admitted operation;
- selected implementation/transport;
- actual attempt;
- result/receipt/publication;
- fallback/retry decisions;
- actor generation.

This allows a reviewer to determine not merely that a command succeeded, but which code and authority performed it.
