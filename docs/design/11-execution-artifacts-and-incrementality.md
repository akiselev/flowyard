# Execution, artifacts, and incrementality

## Reuse Artifactum's model

Artifactum already distinguishes:

- `ActionSpec`: canonical requested computation;
- `ActionKey`: identity of that computation;
- `AttemptRecord`: one execution;
- `Realization`: successful binding to immutable outputs;
- `Artifact`: immutable semantic value;
- source observations and effect receipts;
- pure/reproducible/volatile/effect cache semantics;
- checkpoints, logs, cancellation, and lineage.

Flow should compile suitable function-node invocations into Artifactum actions rather than define a competing cache model.

## Identity mapping

A function-node action identity includes, through Artifactum's canonical projection:

- semantic function ID/version;
- exact implementation/code build identity;
- input artifact IDs;
- canonical reflected arguments;
- relevant actor/service snapshot or version identities;
- declared environment values;
- output contracts/codecs;
- sandbox/network policy;
- platform constraints where output-relevant.

It excludes operational choices that do not change computation, such as retry number, priority, worker identity, and nominal task name.

## Linked implementation identity

A linked function still needs code identity for reusable Artifactum actions. The design requires a build identity covering at least:

- containing executable/library build artifact digest;
- function descriptor fingerprint;
- relevant feature/config build metadata;
- optional source/VCS attestation.

Calling the ordinary Rust function directly outside the Flow action engine does not automatically create a realization. Linking removes transport overhead; it does not eliminate provenance requirements for cached workflow use.

## Remote implementation identity

A remote function/service used in a reproducible action needs a trusted implementation identity or explicit volatile semantics.

Possible evidence includes:

- executable/container digest;
- signed plugin build manifest;
- remote service deployment/version attestation;
- model/index snapshot IDs;
- service-provided deterministic version token.

A mutable endpoint name alone is insufficient for `pure` semantics.

## Function-kind mapping

| Flow function kind | Artifactum behavior |
|---|---|
| `derive(cache = pure)` | Lookup/reuse realization; determinism violations are significant. |
| `derive(cache = reproducible)` | Lookup/reuse; explicit audits may compare repeated results. |
| `function` | Volatile unless caller supplies a stronger explicit policy. |
| `observe` | Always reads external state when invoked; emits observation receipt and immutable artifacts. |
| `effect` | Always executes; emits effect receipt and optional outputs. |

## Runs and target freshness

A Flow run asks the planner to make targets current relative to:

- current actor publication generations;
- current source artifact refs;
- current Flow IR and lock;
- available implementations;
- policy/admission state.

The plan may contain only cache hits. A run remains a meaningful operational record even when no function executes.

## Keyed collections

A collection maps stable logical keys to artifact IDs:

```text
lot:govdeals:1234 -> artifact A
lot:govdeals:1235 -> artifact B
```

When the next snapshot is:

```text
lot:govdeals:1234 -> artifact A   unchanged
lot:govdeals:1235 -> artifact C   changed
lot:govdeals:1236 -> artifact D   added
```

A mapped derivation reuses the action for A, recomputes C and D, and updates downstream aggregate collection structure. This is the desired Salsa-like behavior at artifact scale.

## Open and sealed publication sessions

### Motivation

Spider emits pages during a crawl. Waiting for the full crawl before parsing increases latency, while publishing a partial crawl as current corrupts snapshot semantics.

### Proposed state machine

```text
created -> open -> sealing -> sealed
              \-> aborted
```

An open session has:

- actor/publication identity;
- session/generation ID;
- optional base snapshot;
- keyed item operations;
- checkpoints;
- provisional metadata;
- emitted item events.

### Per-item work

Once an item artifact is committed, mapped downstream actions may execute immediately against it. Their realizations are valid because the item and function action are immutable, even if the enclosing publication later aborts.

### Aggregate gating

Operations requiring the complete collection—collect, full join, rank-all, target publication—wait for the source collection seal unless explicitly declared streaming/partial.

### Seal

Sealing:

1. validates collection key uniqueness and type;
2. applies upserts/removals to the base snapshot;
3. commits the immutable collection artifact;
4. commits receipt/checkpoint metadata;
5. atomically advances the actor publication ref;
6. emits a durable publication event.

### Abort/crash

The named publication remains on its previous sealed value. Item artifacts and successful per-item realizations remain reachable according to retention policy and can be reused after resume.

## Collection deletion semantics

A complete snapshot can represent deletion by absence. Incremental/delta sources may instead emit explicit tombstones until a compacted snapshot is sealed. The publication contract declares:

```text
snapshot_complete
snapshot_partial
append_only
upsert_delete_delta
```

Downstream reconciliation must not infer deletion from absence in a partial snapshot.

## Incremental propagation

The planner propagates change by semantic output identity:

- if a recomputed upstream result has the same artifact ID, downstream actions remain green/reusable;
- if a collection's membership changes, only affected mapped items rerun, while aggregate structure changes;
- if a function implementation/contract changes, affected actions receive new keys;
- if only source formatting/comments change, no action changes;
- if an actor republishes the exact same artifact, a publication event may be recorded but derivations remain current.

This mirrors the useful red/green idea from Salsa while using explicit durable artifacts rather than in-memory query values.

## Leases and retries

The workflow control plane owns:

- claim/admission;
- lease tokens and deadlines;
- heartbeats;
- retry scheduling;
- attempt priority/lane;
- resource/budget reservations;
- stale-attempt rejection.

Artifactum owns attempt and output evidence. A stale workflow attempt cannot advance success/publication even if its external process eventually returns.

Retry policy is bounded and typed. Retrying an observe/effect function requires stronger scrutiny than retrying a pure derivation.

## Checkpoints

Checkpoints are immutable artifacts associated with an action or actor operation. They may include:

- crawler cursor/frontier;
- partial download journal;
- parsed batch offset;
- model-processing position;
- actor publication session state.

A checkpoint is recoverable state, not a successful output. It does not create a realization or advance a publication by itself.

## Logs and diagnostics

Stdout/stderr, structured events, and diagnostic attachments are captured independently from success. A failed attempt remains inspectable without pretending its staged outputs are valid.

## Replay and fork

### Replay

Reconstructs the same:

- function implementation/build;
- canonical arguments;
- input artifacts;
- environment;
- policy/sandbox/network settings;
- available checkpoint.

For observe/effect functions, “replay” defaults to reconstruct/inspect and requires explicit authorization to contact external systems again.

### Fork

Creates a new action request with one or more deliberate changes:

```text
flow attempt fork <id> --set threshold=0.8
flow attempt fork <id> --implementation local:new-model
```

The new identity and diff are explicit.

## Explain

`flow explain` should identify every changed identity component:

```text
Will execute auction.classify for lot:1235.

previous input artifact: sha256:aaa
current input artifact:  sha256:bbb   changed
implementation:          sha256:ccc   unchanged
arguments:               sha256:ddd   unchanged
policy:                   sha256:eee   unchanged

Downstream aggregate `rank` becomes stale.
```

## Effects and exactly-once language

No general runtime can guarantee exactly-once external side effects across crashes and uncertain network responses. The design provides:

- idempotency keys;
- pre-effect reservations;
- receipts;
- typed indeterminate outcomes;
- operator reconciliation;
- external-system query hooks where available.

Documentation and UI must avoid exactly-once claims.

## Data plane versus in-memory optimization

Linking functions allows direct Rust calls and in-process resource reuse. Durable Flow execution still materializes/commits values required for cache, cross-run lineage, or external placement.

The design deliberately does not introduce an implicit transient in-memory graph whose outputs disappear from provenance. A future explicit ephemeral mode may exist, but it must state that it forfeits durability and cross-placement reuse.
