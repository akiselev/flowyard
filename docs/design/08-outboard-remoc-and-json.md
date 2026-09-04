# Outboard, Remoc, and structured JSON

## Objective

Extend Outboard from executable plugin discovery plus one worker protocol into a transport-capable callable-component fabric, while preserving its strongest existing property:

> The executable process is the stable implementation boundary.

The extension must support functions, services, actors, and their descriptors without making Flow a dependency of Outboard.

## Existing strengths to retain

Current Outboard already provides:

- deterministic executable discovery and shadow diagnostics;
- independently versioned framework, interface, and worker protocol requirements;
- manifests and capability properties;
- one-shot and persistent-worker execution modes;
- request IDs, multiplexed Tokio workers, progress/output events, cancellation, ping, and shutdown;
- length-prefixed size-bounded JSON framing;
- OS-string-safe argv transport;
- reusable plugin management/doctor commands;
- black-box conformance testing.

The design extends these mechanisms rather than replacing them.

## Semantic descriptors

Outboard should understand three descriptor classes:

```text
FunctionDescriptor
ServiceDescriptor
ActorDescriptor
```

They are built from Facet/Contract Shape but live in an Outboard contract crate. Flow adds semantics such as derivation caching; Outboard only needs enough function metadata to discover and invoke it.

A plugin manifest advertises:

- plugin identity/build;
- semantic functions/services/actor types;
- descriptor schema versions/fingerprints;
- supported execution modes;
- supported transports;
- optional capabilities;
- health/doctor behavior;
- maximum frame/message limits;
- artifact-reference support.

## Transport negotiation

A persistent host and implementation negotiate a transport after normal Outboard compatibility checks.

Conceptual manifest capabilities:

```text
outboard.transport.json-v2
outboard.transport.remoc-v1
outboard.transport.argv-v1
outboard.artifact-ref-v1
outboard.events-v1
```

Conceptual launch:

```text
plugin __outboard serve --transport remoc-v1
plugin __outboard serve --transport json-v2
```

The launcher supplies the already-authenticated/local process stream or remote byte stream. Remoc itself does not provide authentication or encryption, so transport establishment remains a separate responsibility.

## Resolution and preference

A typical performance preference is:

```text
linked > Remoc > JSON v2 > legacy argv/one-shot
```

The resolver MUST also consider:

- required process isolation;
- trust domain;
- compatible function/service shape;
- streaming/cancellation requirements;
- platform and architecture;
- sandbox and network policy;
- implementation health;
- remote placement;
- explicit user policy.

No fallback is allowed if it drops a required semantic capability. For example, a call requiring authenticated network placement or a bidirectional stream cannot silently fall back to an incompatible one-shot argv implementation.

## Linked transport

Linked implementations register descriptors and local invokers. A linked service client still goes through the actor/service dispatch discipline when state ordering matters, but it does not serialize values.

Linked mode is trusted in-process code. It has no fault or memory isolation and must be labeled accordingly.

## Remoc transport

### Why Remoc fits

Remoc's remote trait calling provides:

- generated clients and server forms;
- concurrent calls and pipelining;
- cancellation behavior;
- request monitoring and tracing support;
- remote channels and objects;
- named request arguments and additive compatibility patterns;
- local client use;
- a request receiver that can be processed as messages.

The request receiver is particularly useful for actors: the Outboard/actor runtime can route requests into the actor mailbox instead of handing Remoc direct mutable access to actor state.

### Boundary rule

Remoc is a generated transport adapter. It is not the source of semantic IDs, versions, docs, or Contract Shape.

The baseline service/function contract requires Facet/Contract types. Enabling a Remoc adapter additionally requires the relevant values to satisfy Remoc/Serde remote-send constraints. A plugin may therefore support JSON but not Remoc for a particular contract; the manifest reports this honestly.

### Rich features

Portable wrappers should cover common rich behavior:

- `RemoteStream<T>`;
- `Subscription<T>`;
- `ServiceRef<S>`;
- `ActorRef<S>`;
- cancellation/progress.

The Remoc adapter maps these to native remote channels/objects. The JSON adapter maps them to IDs and event frames.

Raw Remoc channel/client types MAY appear only in a method explicitly marked Remoc-only. Such a method advertises a transport requirement and receives no implied language-neutral fallback.

## Structured JSON v2

The current JSON worker protocol is argv/command-oriented. JSON v2 should add typed structured operations while retaining legacy compatibility.

Illustrative host frames:

```json
{"type":"hello","protocol":"2.0","transports":["json-v2"],"contracts":[...]}
{"type":"describe","request_id":1,"kind":"function","id":"auction.parse"}
{"type":"invoke_function","request_id":2,"function":"auction.parse","version":"1.0.0","args":{...}}
{"type":"call_service","request_id":3,"actor":"govdeals","service":"spider.crawler","method":"refresh","args":{...}}
{"type":"subscribe","request_id":4,"stream":"..."}
{"type":"cancel","request_id":2}
{"type":"shutdown"}
```

Illustrative plugin events:

```json
{"type":"started","request_id":2,"attempt":"..."}
{"type":"progress","request_id":2,"fraction":0.4,"message":"120/300 pages"}
{"type":"item_published","request_id":2,"port":"pages","key":"...","artifact":"artifact:sha256:..."}
{"type":"diagnostic","request_id":2,"diagnostic":{...}}
{"type":"finished","request_id":2,"result":{...}}
```

Frames remain length-prefixed and size-bounded on binary streams. JSONL may be used for explicitly line-oriented stdio agent interfaces, but the worker protocol does not need to abandon length framing.

## Contract values

JSON requests represent semantic wrappers explicitly:

```json
{
  "listing": {
    "$artifact": "artifact:sha256:...",
    "type": "auction.listing@2"
  },
  "threshold": 0.72
}
```

Other reserved forms can represent collections, actor refs, service refs, secret refs, paths, and subscriptions. The reserved envelope syntax is versioned and validated against Contract Shape.

## Large data

The current Outboard principle remains correct: JSON/Remoc control messages are not bulk blob transport.

Bulk data uses:

- Artifactum IDs with verified local/remote materialization;
- read-only sandbox paths;
- file descriptors/handles where safely supported;
- shared memory as an explicit optional transport capability;
- dedicated byte streams for cases that cannot be artifactized first.

A plugin never gains authority to claim that arbitrary bytes already have a particular Artifactum identity. The host verifies/imports outputs.

## Process and daemon sessions

### Ephemeral worker

The host launches an executable, negotiates a transport, serves one or more calls, then terminates it.

### Persistent local worker

Daemonkit owns startup, authenticated local attachment, readiness, generation, and cleanup. Outboard owns the application session and transport negotiation over the authenticated stream.

### Remote worker

A future remote placement layer establishes an authenticated/encrypted byte stream, identifies the peer, and then uses the same Outboard transport negotiation. Network discovery and trust are intentionally outside Remoc itself.

## Function invocation over Remoc

Functions do not naturally require one generated trait each. Two viable adapters exist:

1. a generic typed-erased Remoc call service carrying contract values; or
2. generated per-function/per-module Remoc methods.

The design preference is hybrid:

- ordinary functions use a generic function transport with generated typed wrappers at the caller;
- services use generated trait-native Remoc adapters;
- large values remain artifact refs;
- hot linked calls bypass both.

This avoids generating enormous remote traits while retaining Remoc's strongest service features.

## Actor addressing

Structured calls identify:

```text
actor type
instance key
requested generation policy
service ID/version
method ID/name
```

The host resolves the actor instance locally or remotely. A stale pinned generation returns a typed generation error; it does not silently call the replacement.

## Failure model

Failures are classified independently:

- definition/contract mismatch;
- implementation resolution failure;
- transport establishment failure;
- protocol violation;
- actor unavailable/generation changed;
- call rejected by policy/admission;
- domain error;
- cancellation;
- timeout;
- worker exit/crash;
- indeterminate external effect.

Fallback may occur only before semantic call admission unless the operation is explicitly idempotent and retry policy authorizes re-execution.

## Compatibility with existing Outboard

The extension should be additive:

- current manifests remain readable;
- current `__outboard manifest|doctor|cli-schema|ping|serve` remains meaningful;
- current argv invocation remains available;
- JSON v1 and v2 protocols have independent version negotiation;
- existing plugins need not depend on Facet unless they export structured contracts;
- current `PluginCommands` management UX remains usable and can surface new descriptors.

## Conformance

Every transport implementation should run the same semantic conformance suite:

- descriptor equality;
- required/default argument behavior;
- result/error shape;
- cancellation claims;
- progress/event ordering;
- frame/message limits;
- non-UTF-8 path behavior where supported;
- actor generation behavior;
- stream completion and disconnect;
- malformed and adversarial peer behavior.

Transport-specific tests are additional, not substitutes.
