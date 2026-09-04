# Worked example: PartFoundry

This example preserves PartFoundry's evidence and rights boundaries while expressing shared orchestration through actors, functions, and Flow IR.

## Domain functions

```rust
#[flow::observe(id = "foundry.acquire-source", version = 3)]
async fn acquire_source(
    context: &ObserveContext,
    request: SourceAcquisitionRequest,
) -> Result<SourceAcquisitionResult, SourceError>;

#[flow::derive(id = "foundry.parse-source", version = 2, cache = "pure")]
async fn parse_source(
    input: Artifact<FetchedSourceBundle>,
    parser: ParserBinding,
) -> Result<Artifact<BronzeBatch>, ParseError>;

#[flow::derive(id = "foundry.reconcile", version = 1, cache = "pure")]
async fn reconcile(
    observations: Collection<BronzeBatch>,
    policy: Artifact<ReconciliationPolicy>,
) -> Result<ReconciliationOutputs, ReconcileError>;

#[flow::effect(id = "foundry.promote", version = 1)]
async fn promote(
    request: PromotionRequest,
) -> Result<PromotionReceipt, PromotionError>;
```

`promote` is effectful because it advances a mutable application projection/ref after policy evaluation. Its evidence inputs and receipt remain immutable.

## Source actor service

```rust
#[outboard::service(id = "foundry.source-control", version = 1)]
pub trait SourceControlService {
    async fn status(&self) -> Result<SourceStatus, SourceError>;
    async fn refresh(&self, request: RefreshRequest)
        -> Result<SourceOperation, SourceError>;
    async fn checkpoint(&self) -> Result<Artifact<SourceCheckpoint>, SourceError>;
}
```

A generic/specialized source actor owns:

- exact reviewed profile/policy/authority/plugin bindings;
- checkpoints and refresh/backoff;
- acquisition admission and budget;
- invocation of networked observe function;
- networkless parser function;
- sealed publication of Bronze batches and receipts.

## KDL

```kdl
flow-ir 1

project "partfoundry" {
    actor "microchip" use="foundry.source-actor@^1" {
        config {
            source-key "manufacturer.microchip.onlinedocs"
            profile (path)"config/sources/manufacturer.microchip.onlinedocs.profile.kdl"
            policy (path)"config/sources/policy.microchip.onlinedocs-private.kdl"
            authority (path)"config/sources/authority.microchip.kdl"
            plugin "foundry-source-microchip-onlinedocs@^1"
            refresh (duration)"24h"
        }

        expect-publication "bronze" type="Collection<foundry.bronze-batch@2>"
        expect-publication "receipts" type="Collection<foundry.source-run-receipt@3>"
    }

    actor "ti" use="foundry.source-actor@^1" {
        config {
            source-key "manufacturer.ti.catalog"
            profile (path)"config/sources/manufacturer.ti.catalog.profile.kdl"
            policy (path)"config/sources/policy.ti.catalog.kdl"
            authority (path)"config/sources/authority.ti.kdl"
            plugin "foundry-source-ti@^1"
            refresh (duration)"24h"
        }

        expect-publication "bronze" type="Collection<foundry.bronze-batch@2>"
        expect-publication "receipts" type="Collection<foundry.source-run-receipt@3>"
    }

    flow "corpus" {
        source "microchip" from="actor.microchip.bronze"
        source "ti" from="actor.ti.bronze"

        collect "bronze" {
            input "microchip" from="microchip"
            input "ti" from="ti"
        }

        call "reconcile" use="foundry.reconcile@^1" {
            input "observations" from="bronze.collection"
            input "policy" from="@ref/reconciliation-policy"
        }

        call "assess" use="foundry.assess-candidates@^1" {
            input "candidates" from="reconcile.candidates"
            input "authority" from="@ref/current-authority-policies"
        }

        call "gold-plan" use="foundry.plan-promotion@^1" {
            input "assessments" from="assess.results"
        }

        target "silver" from="reconcile.silver"
        target "gold-candidates" from="gold-plan.requests"
    }

    flow "promote-gold" {
        source "requests" from="flow.corpus.gold-candidates"

        map "promote" use="foundry.promote@^1" {
            each "request" from="requests.items"
        }

        target "receipts" from="promote.results"
    }
}
```

Separating the promotion flow makes effect admission obvious. A normal corpus reconciliation cannot silently advance Gold.

## Exact binding identity

A source refresh operation commits:

```text
source profile digest
source policy digest
source authority-policy digest
plugin executable/build digest
input/checkpoint digest
observation time
Artifactum store identity
bounded retry/budget policy
```

The actor revalidates these bindings after claim and before external access. A stale attempt cannot publish success or advance source refs.

## Artifact ownership

```text
Fetched response bytes           Artifactum
Fetch/source-run receipt          Artifactum
Bronze Parquet batch              Artifactum
Silver output                     Artifactum
Actor/control operation state     workflow/actor control store
Gold relational projection        PartFoundry application database
Rights/authority decisions        PartFoundry domain contracts
```

## Agent queries

```text
flow actor status microchip --format json
flow actor call microchip foundry.source-control refresh \
  --args-json '{"force":false}' --format jsonl

flow plan corpus/silver --format json
flow explain corpus/reconcile --format json
flow lineage @silver --format json
```

## What the generic framework does not decide

- whether a source is authoritative for a claim;
- whether retained bytes may be published, embedded, trained on, or exported;
- whether normalized MPNs identify the same part;
- whether a Bronze observation may contribute to Silver/Gold;
- how document applicability works;
- whether credentials/agreement terms permit an operation.

Those remain versioned PartFoundry contracts evaluated through generic policy/admission hooks.
