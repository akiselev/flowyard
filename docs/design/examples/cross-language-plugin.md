# Worked example: cross-language structured JSON plugin

## Goal

A non-Rust executable should implement the same semantic function/service contracts without understanding Facet or Remoc internals.

Facet-derived Contract Shapes are exported in a portable manifest/schema. The plugin implements Outboard's structured JSON v2 protocol and uses Artifactum references for bulk values.

## Example function contract

```json
{
  "schema": "outboard.function-contract/v1",
  "id": "auction.enrich",
  "version": "1.0.0",
  "kind": "function",
  "args": {
    "type": "object",
    "fields": {
      "listing": {
        "type": "artifact",
        "artifact_type": "auction.listing@2",
        "required": true
      },
      "model": {
        "type": "string",
        "required": false,
        "default": "gemini-flash"
      }
    }
  },
  "result": {
    "type": "artifact",
    "artifact_type": "auction.enrichment@1"
  },
  "capabilities": [
    "network:model-provider",
    "secret:model-api-key",
    "outboard.events-v1",
    "outboard.artifact-ref-v1"
  ]
}
```

The actual portable Contract Shape may be richer than JSON Schema. This example shows the information a language plugin needs.

## Outboard manifest excerpt

```json
{
  "manifest_version": 2,
  "plugin": {
    "namespace": "auction",
    "kind": "function",
    "name": "enrich-python",
    "version": "0.4.0"
  },
  "functions": [
    {
      "id": "auction.enrich",
      "version": "1.0.0",
      "contract_fingerprint": "sha256:..."
    }
  ],
  "transports": ["json-v2"],
  "execution": ["worker"],
  "capabilities": ["outboard.artifact-ref-v1", "outboard.events-v1"]
}
```

## Worker invocation

```text
auction-function-enrich-python __outboard serve --transport json-v2
```

After length-framed handshake, the host sends:

```json
{
  "type": "invoke_function",
  "request_id": 42,
  "function": "auction.enrich",
  "version": "1.0.0",
  "args": {
    "listing": {
      "$artifact": "artifact:sha256:abc...",
      "type": "auction.listing@2"
    },
    "model": "gemini-flash"
  },
  "context": {
    "attempt": "attempt:...",
    "deadline": "2026-09-04T18:00:00Z",
    "secrets": {
      "model-api-key": "capability:secret-handle-17"
    }
  }
}
```

The secret handle is resolved through a constrained side channel/environment or host API; the raw secret is not present in descriptors/logs.

## Plugin events

```json
{"type":"started","request_id":42}
{"type":"progress","request_id":42,"fraction":0.1,"message":"loading images"}
{"type":"diagnostic","request_id":42,"diagnostic":{"code":"image.low_resolution","severity":"warning","message":"..."}}
```

The plugin writes its result into a host-provided output staging path or artifact upload stream and returns a claim:

```json
{
  "type": "finished",
  "request_id": 42,
  "result": {
    "$staged_output": "enrichment",
    "type": "auction.enrichment@1",
    "declared_sha256": "..."
  }
}
```

The host verifies the digest/type and commits the Artifactum artifact. The plugin cannot directly mint a trusted artifact ID.

## Cancellation

```json
{"type":"cancel","request_id":42,"reason":"caller_cancelled"}
```

The plugin responds with a terminal cancellation event after cooperative cleanup. If it ignores cancellation, the host may terminate the process according to policy.

## Service call example

A Python actor/service host could receive:

```json
{
  "type": "call_service",
  "request_id": 61,
  "actor": {
    "type": "pricing.index@1",
    "instance": "default",
    "generation": "follow-current"
  },
  "service": "auction.pricing@1",
  "method": "estimate",
  "args": {
    "request": {
      "listing": {"$artifact":"artifact:sha256:...","type":"auction.listing@2"}
    }
  }
}
```

Long-lived streams use a returned subscription ID and normal event frames. The Rust Remoc adapter may use native remote channels for the same semantic `RemoteStream<T>`.

## Conformance

The plugin package should be testable without the host application:

```text
flow plugin conformance ./auction-function-enrich-python
flow plugin describe ./auction-function-enrich-python --format json
flow plugin doctor ./auction-function-enrich-python --deep
```

Conformance validates:

- manifest/descriptor fingerprint;
- default and required fields;
- malformed requests;
- event ordering and terminal result;
- frame/output limits;
- cancellation;
- artifact staging and digest mismatch;
- structured error contract.

## What is intentionally absent

- no generated Rust client required;
- no Remoc dependency;
- no direct Artifactum database write;
- no dynamic-library ABI;
- no need to compile the main orchestrator;
- no shell parsing of ad hoc stdout as the result protocol.
