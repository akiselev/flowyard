# Restate

Researched: 2026-09-16. Scope: docs.restate.dev (live, September 2026; server 1.7.x, latest release v1.7.10 published 2026-09-14); restate-sdk crate 0.12.0 (published 2026-09-02) read from github.com/restatedev/sdk-rust `main` (`src/lib.rs`, `src/context/mod.rs`, `src/context/run.rs`, `src/context/select.rs`, `src/context/select_any.rs`, `src/configuration.rs`, README); Restate v1.5.0 release notes (2025-09-16). Rust docs pages under docs.restate.dev/develop/rust/* now redirect to docs.rs; the SDK source doc comments are the primary Rust reference.

## Summary
Restate is a single-binary durable-execution server written in Rust that invokes user handlers over HTTP and records every context operation in a journal, replaying the journal on retries so handler code resumes where it left off. Users write Services (stateless), Virtual Objects (keyed, single-writer) and Workflows (keyed, `run` executes exactly once per ID) in an SDK; the server owns retries, timers, state, and routing. It optimizes for low-latency RPC-style durable functions, microservice orchestration and event processing across a fleet of stateless service replicas, with the server as the only stateful component. The Rust SDK is 0.x (0.12.0), small (85 stars) but actively released; the Restate server is 4.4k stars.

## Deployment and process model
- Server/daemon: `restate-server` (brew/npm/docker/binary). "Restate is a distributed architecture in a box (single binary)". Default ports: 8080 ingress, 9070 admin API + bundled UI, 5122 node-to-node. "State and invocations are stored in the `restate-data` directory" (docs.restate.dev/develop/local_dev). In single-node mode "you are hosting all the essential features in a single process" (references/architecture).
- Services are user processes exposing an HTTP endpoint: `HttpServer::new(Endpoint::builder().bind(MyService).build()).listen_and_serve("0.0.0.0:9080".parse().unwrap()).await;` then `restate deployments register http://host:9080`. The server pushes invocations to the endpoint ("invokes your handler code via a bidirectional stream"); Lambda is also supported (SDK 0.7.0+).
- What must run: the Restate server AND a reachable endpoint of the deployment the invocation is pinned to. There is no library/embedded mode; a one-shot process cannot make progress on its own (Q3: daemon).

## Execution model
Journal replay: "Restate tracks every step of your code execution in a journal ... Restate replays the journal, skipping completed steps and resuming from exactly where it left off." (concepts/durable_execution). Each context call (call, send, sleep, run, get/set state, awakeable, promise) becomes a journal entry; on retry the SDK replays entries in order and returns recorded results.

Determinism rules, verbatim from `restate_sdk::context::ContextSideEffects` (src/context/mod.rs):
> "Restate uses an execution log for replay after failures and suspensions. This means that non-deterministic results (e.g. database responses, UUID generation) need to be stored in the execution log."
> "You can store the result of a (non-deterministic) operation in the Restate execution log (e.g. database requests, HTTP calls, etc). Restate replays the result instead of re-executing the operation on retries."
> "You cannot use the Restate context within `ctx.run`. This includes actions such as getting state, calling another service, and nesting other journaled actions."
> "**Caution: Immediately await journaled actions:** Always immediately await `ctx.run`, before doing any other context calls. If not, you might bump into non-determinism errors during replay, because the journaled result can get interleaved with the other context calls in the journal in a non-deterministic way."

Deterministic helpers: `random_seed()` "Return a random seed inherently predictable, based on the invocation id ... stable during the invocation lifecycle, thus across retries"; `rand()`; `rand_uuid()` ("You can use these UUIDs to generate stable idempotency keys ... Do not use this in cryptographic contexts."). Combinators: `restate_sdk::select!` ("Note: This API is experimental and subject to changes."), `DurableFuturesUnordered` ("yields results as they complete ... Each future is assigned a stable index when pushed"), durable `map`/`map_ok`/`map_err` (0.11.0). Ordinary `tokio::select!`/`join!` over durable futures are not documented as safe; the SDK provides its own.

## Step and operation identity
- Identity is positional in the journal (entry index); `ctx.run` names are "used mainly for observability" (`RunFuture::name`; the required name parameter was removed in SDK 0.3.0).
- Mismatch: replay with different code yields "non-determinism errors" and the invocation keeps retrying against the new code: "in-flight invocations might keep failing with non-determinism errors", cleaned up with `restate invocations kill` (services/versioning). Resuming a paused invocation on another deployment "risks non-determinism errors if business logic differs" (managing-invocations).
- Unreached recorded entries (Q1): not documented explicitly. Because matching is positional, a journal entry that the new code never emits surfaces as a mismatch at the first position where entry types differ; if the new code simply finishes earlier, behavior is unverified. The documented mitigation is never to change code under a registered deployment (see Versioning).

## Persistence schema
- Store: proprietary. "New events (invocations, journal entries, state updates, ...) are persisted in an embedded replicated log (called Bifrost). From there, events move to state indexes in RocksDB, which are periodically snapshotted to an object store." "The processor leader maintains a full cache of the partition's materialized state in RocksDB ... This cache is derivative—it can always be rebuilt from the log." (references/architecture). `bifrost.default-provider` selects `local` (RocksDB-backed loglet, single node) or `replicated`; `base-dir` default `restate-data`; `rocksdb-disable-wal-fsync` controls fsync per WAL batch (references/server-config).
- No SQL DDL; introspection exposes virtual tables via `restate sql`: `sys_invocation`, `sys_journal`, `sys_keyed_service_status`, `sys_deployment`, `state` (services/introspection). Journal retention after completion defaults to 24 h (1.5.0+, configurable `journal_retention`).
- Step outputs live inline in journal entries (serialized by the SDK's `serde` module). Size: a search lead points at `worker.invoker.message_size_limit` (default 32 MiB) as the per-message cap (docs.restate.dev/references/errors); not fetched verbatim — unverified. Large page bodies belong in external storage with references in the journal; the docs give no claim-check helper.

## Crash recovery of in-flight work
- Service process dies mid-handler: the server sees the stream close, marks the invocation for retry, and re-invokes the same deployment; replay skips completed entries. A `ctx.run` whose result was not yet journaled re-executes (at-least-once), so run closures must be idempotent (docs assume this; the Sagas guide covers compensation).
- Server timeouts: `inactivity_timeout` (default 1 m) "How long an invocation may stay in-flight without making progress before Restate asks it to suspend gracefully"; `abort_timeout` (default 10 m) "How long Restate waits after the inactivity timeout before forcibly aborting the invocation (killing the handler task)" (src/configuration.rs). No heartbeat API; progress is measured by new journal entries.
- Server crash: recovered from the log (+ snapshots); "Restate Server crashes (via high-availability clustering or persistent disk recovery)" (concepts/durable_execution).
- Declarable modes: retryable (default, everything) vs `TerminalError`; `on_max_attempts = "pause"` leaves the invocation for a human to inspect and `restate invocations resume`. There is no "uncertain effect — stop for reconciliation" classification other than throwing TerminalError yourself.

## Retries, backoff, timeouts
- Default: "Restate assumes by default that all errors are transient errors and therefore retryable" ... "Restate retries failures infinitely. Use `TerminalError` to stop retries." (guides/error-handling; lib.rs).
- Invocation retry policy (Restate 1.5.0, 2025-09-16): "The new retry policy will now, by default, **pause** an invocation when **max-attempts** is reached." Fields from the services/configuration page: `initial-interval` 50ms, `exponentiation-factor` 2.0, `max-interval` 60s, `max-attempts` (page lists 70; the error-handling guide describes `max-attempts = "unlimited"` — verify against your server version), `on-max-attempts` `pause` (default) or `kill`. Configurable globally, per service, per handler, via CLI (`restate services config edit`), UI, or SDK attributes:
```rust
#[restate_sdk::service(
    inactivity_timeout = "10m",
    abort_timeout = "1m",
    idempotency_retention = "1 day",
    invocation_retry_policy(
        initial_interval = "100ms",
        factor = 2.0,
        max_interval = "3s",
        max_attempts = 10,
        on_max_attempts = "pause",
    ),
)]
```
- Per-step policy on `ctx.run` (`RunRetryPolicy`): `initial_delay`, `exponentiation_factor`, `max_delay`, `max_attempts` ("**Note:** The number of actual retries may be higher than the provided value ... Infinite retries if this field and `max_duration` are unset."), `max_duration`. "If you set a maximum number of attempts, then the `ctx.run` block will fail with a [TerminalError] once the retries are exhausted."
- Persistence: retry state is server-side; `restate invocations list --status backing-off` shows retry counts and last failure. Jitter: unverified (not mentioned). Custom delay from a `Retry-After` header: `RetryableError` with explicit delay (documented for TS; Rust equivalent unverified).
- Retries are per invocation (the whole handler replays) with per-`run` overrides — a hybrid of task-level and step-level retry.

## Flow control
- Virtual Objects are the primary primitive: "At most one handler with write access can run at a time per object key"; "Invocations to a Virtual Object are executed serially. Invocations will execute in the same order in which they arrive at Restate." Keyed by object key, persisted (queue lives in the server), cross-process.
- Flow control (Restate 1.7, "Starting with Restate 1.7.3, you can enable flow control on an existing cluster"): concurrency limits per *scope* with optional two-level *limit keys*: "Matching invocations are throttled to the configured concurrency and held in their queue until a slot frees up." Rules are "defined in a cluster-wide rule book" (`restate rules set "checkout" --concurrency 50`), i.e. persisted and cross-process. "A scope ... becomes part of the identity of every invocation and resource inside it"; "A limit key only influences concurrency. It is not part of an invocation's identity." Rust SDK: "Scope and limit key: requires Restate >= 1.7 with sdk-rust >= 0.11" (README).
- Rate limiting (per-second) is not built in: the guides/rate-limiting page implements a token bucket as a Virtual Object using durable state and timers (`wait`, `reserve`, `allow`, `setRate`), keyed by limiter name.
- Footgun: "Request-response calls to Virtual Objects can lead to deadlocks, in which the Virtual Object remains locked and can't process any more requests" (cross or cyclic calls); resolved by cancelling invocations from the CLI.

## Fan-out, child workflows, map
- Calls: `ctx.service_client::<MyServiceClient>().handle().call().await`, `.send()`, `.send_after(Duration)`; `ctx.object_client::<..>("key")`, `ctx.workflow_client`. Every call is a child invocation with its own invocation id; ingress calls can carry an `idempotency_key`.
- Parallelism: push durable futures into `DurableFuturesUnordered` and consume `(index, result)` in completion order (stable index correlates to input), or `select!`. There is no "record the work set then map" primitive; the work set is implicit in the sequence of journaled call entries, and ordering/failure policy is user code.
- Workflows: "The `run` handler executes exactly once per workflow instance"; shared handlers (`SharedWorkflowContext`) can `resolve_promise`/`peek_promise` and query state concurrently. `workflow_completion_retention` (default 24 h) bounds how long the result/state remain.
- Singleton per key (Q2): yes, by construction — Virtual Object handlers with `ObjectContext` are exclusive per key; a Workflow's `run` executes exactly once per workflow ID (a second start with the same ID is rejected/deduplicated; retention applies).

## Versioning and code change policy
Verbatim (docs.restate.dev/services/versioning): "when you deploy a version of your code, you give it an immutable, unique endpoint and register it with Restate." ... "requests start and end on the same version, by sending any retry attempts always to the same endpoint." In-place changes that "reorder Restate SDK operations" or "add or remove SDK operations in the execution path" can cause "in-flight invocations to fail when they replay with the updated code." New version: `restate deployments register http://greeter-v2/`; "Restate automatically routes new requests to the latest deployment. Existing requests continue on the original deployment." Local dev uses `--force` to re-register the same URL, after which "in-flight invocations might keep failing with non-determinism errors" -> `restate invocations kill`. Remove old deployments only after `restate deployment describe <id> --extra` shows no active invocations. Interface rules: "Adding handlers is allowed, but removing or renaming them is not." (`--breaking` to override). No version markers/patching API; the unit of compatibility is the deployment.

## Schedules, timers, missed runs
- `ctx.sleep(Duration)` is a durable timer computed by the SDK and fired by the server ("The Restate SDK calculates the wake-up time ... make sure that the system clock of the Restate Server and the SDK ... are synchronized"). "Virtual Objects only process a single invocation at a time, so the Virtual Object will be blocked while sleeping." Timers survive restarts.
- Delayed sends: `client.handle().send_after(Duration::from_secs(60))` — "The calling handler can finish execution and complete the invocation without waiting for the delay to finish".
- Cron: "Restate does not yet include native cron support, but you can implement your own cron scheduler" (develop/ts/durable-timers). The guides/cron pattern is a `CronJob` Virtual Object that computes the next fire time and `send_after`s itself. Missed runs after downtime: persisted delayed invocations fire once each when the server is back, then compute the next tick from the current time (one current run per job, not a burst — inferred from the pattern). Overlap: the guide warns "you would have concurrent executions of the same cron job (one retrying and the other starting up)" unless the job checks/cancels the previous execution.

## Cancellation and late results
- `restate invocations cancel <id>`: "propagates recursively through the call graph", reaching leaves first; in the handler a `TerminalError` surfaces "at the next await point—operations that wait for a result, such as `ctx.run()`, call responses, sleeps, awakeable results, etc." so compensation can run (sagas). `select!` has an `on_cancel` arm.
- `kill`: "immediately stops all calls in the invocation tree without executing compensation logic", "a last resort". `purge` removes completed invocations; `pause`/`resume` (optionally `--deployment`).
- Late results: a `ctx.run` closure that completes after cancellation returns into a handler that already sees TerminalError at its next context call; the journal keeps the entry. No further doc statement (unverified).

## Observability and tooling
- CLI: `restate invocations list [--status backing-off|--service X --key K|--virtual-objects-only]`, `describe <id>` (status, journal entries, last modified, caller, trace id), `cancel/kill/purge/pause/resume`, `restate kv get/edit`, `restate sql`, `restate deployments register/describe`, `restate services config edit`, `restate rules set`. Statuses: "pending, ready, running, backing-off, retrying due to a failure, suspended, waiting on some external input, or completed."
- UI bundled at :9070 ("manage, debug, and configure your applications"); OpenTelemetry tracing; "Restart-as-new functionality for failed/succeeded invocations" and "Immediate retry option" (1.5.0).

## API ergonomics
Minimal complete example, verbatim from github.com/restatedev/sdk-rust/blob/main/README.md:
```rust
use restate_sdk::prelude::*;

struct Greeter;

#[service]
impl Greeter {
    #[handler]
    async fn greet(&self, _ctx: Context<'_>, name: String) -> HandlerResult<String> {
        Ok(format!("Greetings {name}"))
    }
}

#[tokio::main]
async fn main() {
    // To enable logging/tracing
    // tracing_subscriber::fmt::init();
    HttpServer::new(
        Endpoint::builder()
            .bind(Greeter)
            .build(),
    )
    .listen_and_serve("0.0.0.0:9080".parse().unwrap())
    .await;
}
```
Journaled step and virtual object, verbatim from src/lib.rs and src/context/mod.rs:
```rust
let response = ctx.run(|| do_db_request()).await?;
```
```rust
#[restate_sdk::object]
impl MyVirtualObject {
    #[handler]
    async fn my_handler(&self, ctx: ObjectContext<'_>, greeting: String) -> Result<String, HandlerError> {
        Ok(format!("{} {}", greeting, ctx.key()))
    }
    #[handler]
    async fn my_concurrent_handler(&self, ctx: SharedObjectContext<'_>, greeting: String) -> Result<String, HandlerError> {
        Ok(format!("{} {}", greeting, ctx.key()))
    }
}
```
Delightful:
- Exclusivity is a type: `ObjectContext` (exclusive, may write state) vs `SharedObjectContext` (concurrent, read-only); `WorkflowContext` vs `SharedWorkflowContext`. "The shared/exclusive kind is inferred from the context type."
- Generated typed clients for durable calls (`MyServiceClient`) and, since 0.12.0, typed ingress clients (`GreeterIngressClient`) with `.idempotency_key(..)`, `.call()`, `.send()`, `.send_after(..)`.
- Configuration as attribute arguments on `#[service]`/`#[handler]` (timeouts, retention, retry policy) plus programmatic `ServiceOptions`.
- Single-binary server with bundled UI and SQL introspection; `restate-sdk-testcontainers` for integration tests.
- `ctx.run` takes a plain closure returning a future; no explicit step names required.
Painful / footguns:
- The `ctx.run` closure restrictions and the "immediately await" rule (verbatim above); interleaving durable futures outside the SDK's combinators is undefined.
- Virtual Object deadlocks with request-response calls between keyed objects.
- Local dev loop needs a running server plus `restate deployments register --force`, which "might keep failing with non-determinism errors" for in-flight work.
- `select!` is "experimental and subject to changes"; SDK is 0.x with breaking changes each minor (MSRV 1.90 as of 0.9.0).
- Rust docs pages on docs.restate.dev redirect to docs.rs; feature parity examples (durable steps, timers, cron) are documented for TS/Java/Python/Go.
- Services must be reachable HTTP servers; the server pushes to them. Not a library.
Scraping pipeline sketch: `Source` as a Virtual Object keyed by source id (gives the singleton and a per-source queue); `refresh` handler: `ctx.run(|| fetch(url))` per page, immediately awaited, or fan out by `send`ing `fetch_page` to a `Fetcher` object keyed by host (serializes per host) and collecting via `DurableFuturesUnordered`; parse in-handler if deterministic or in another `ctx.run`; enrich via a service call with an idempotency key; publish in a final `ctx.run` guarded by a `RunRetryPolicy` with `max_attempts`; a `CronJob` object to schedule refreshes. User plumbing: external blob storage for page bodies, host rate limiter object, work-set bookkeeping (which pages belong to this refresh), compensation on TerminalError, deployment registration per build.

## Known limitations
- No native cron (develop/ts/durable-timers).
- Immutable deployments; in-place code changes cause non-determinism errors (services/versioning).
- Virtual Objects block while sleeping/awaiting awakeables/signals (ContextTimers, ContextAwakeables docs); deadlocks possible (ContextClient docs).
- `select!` experimental (src/context/select.rs).
- Retry-count semantics of `ctx.run` are approximate ("may be higher than the provided value").
- Flow control requires Restate >= 1.7.3 and enabling vqueues (services/flow-control).
- Servers using old SDKs (Rust < 0.4) cannot register on 1.5+ (v1.5.0 release notes).

## Relevance to Flowyard
- Borrow:
  - Context types that encode exclusivity and capability (`ObjectContext`/`SharedObjectContext`); Flowyard's `WorkflowContext` vs `ActivityContext` split is the same idea.
  - Deterministic `rand`/`uuid` seeded from the run id for stable idempotency keys.
  - Two-level retry: invocation-level policy with `on_max_attempts = pause` (leave it for a human) plus per-step `RunRetryPolicy` with `max_attempts` and `max_duration`.
  - `pause`/`resume`/`restart-as-new`/`kill` as distinct operator verbs; introspection via SQL over run/journal tables.
  - Deployment pinning: a run is coupled to the build that started it; new builds serve new runs.
  - Flow control keyed by scope and hierarchical limit key (host/path) as the shape for per-host concurrency limits.
- Avoid:
  - Positional journal identity plus closure-based steps with no name; the "immediately await" footgun follows from it.
  - Push-based server->service invocation and the mandatory HTTP endpoint.
  - Infinite retries by default for scrapers hitting rate limits.
  - Implicit work sets in fan-out.
- Answers to open questions:
  - Q1: not documented; positional matching means unreached entries surface as non-determinism errors at the first mismatch, and the fix is kill/resume on a compatible deployment (services/versioning; managing-invocations) — partially unverified.
  - Q2: yes; Virtual Object key = single writer; Workflow `run` exactly once per ID (concepts/services; lib.rs).
  - Q3: daemon (restate-server) plus always-reachable service endpoints (local_dev; architecture).
  - Q4: `ctx.run` executes inline in the handler future; calls to other handlers go through the server's persisted per-partition queues (ContextSideEffects; ContextClient; architecture).
  - Q5: persisted, cluster-wide concurrency limits keyed by scope + limit key (1.7); per-second rate limits are user-built Virtual Objects keyed by limiter name (services/flow-control; guides/rate-limiting).
  - Q6: anything deterministic (pure compute) needs no `run`; the docs draw the line at "non-deterministic results (e.g. database responses, UUID generation)" (ContextSideEffects).
  - Q7: inline in the journal; message size limit around 32 MiB per invoker message (unverified); no built-in claim check.
  - Q8: immutable deployment per build, no version attribute, no warn-and-continue; `--force` is the documented footgun (services/versioning).
  - Q9: server-side attempt counter and backoff state visible as `backing-off`; policy declared per service/handler/run; jitter unverified (services/configuration; introspection).
  - Q10: no native schedules; the cron pattern yields one catch-up tick per job after downtime and warns about overlap (guides/cron; develop/ts/durable-timers).

## Sources
- https://docs.rs/restate-sdk/latest/restate_sdk/ — crate overview (0.12.0)
- https://docs.rs/restate-sdk/latest/restate_sdk/context/trait.ContextSideEffects.html — run() docs
- https://raw.githubusercontent.com/restatedev/sdk-rust/main/src/lib.rs — SDK overview, examples
- https://raw.githubusercontent.com/restatedev/sdk-rust/main/src/context/mod.rs — context traits, warnings, ordering/deadlocks
- https://raw.githubusercontent.com/restatedev/sdk-rust/main/src/context/run.rs — RunRetryPolicy
- https://raw.githubusercontent.com/restatedev/sdk-rust/main/src/context/select.rs, select_any.rs — select!, DurableFuturesUnordered
- https://raw.githubusercontent.com/restatedev/sdk-rust/main/src/configuration.rs — attribute configuration, retry policy
- https://raw.githubusercontent.com/restatedev/sdk-rust/main/README.md — minimal example, ingress client, flow-control version note
- https://docs.restate.dev/changelog/rust-sdk — version history
- https://docs.restate.dev/concepts/services — service types, single writer per key
- https://docs.restate.dev/concepts/durable_execution — journal/replay
- https://docs.restate.dev/operate/versioning (same content as /services/versioning) — immutable deployments
- https://docs.restate.dev/operate/introspection — CLI, SQL tables
- https://docs.restate.dev/develop/local_dev — single binary, ports, data dir
- https://docs.restate.dev/references/server-config — rocksdb, bifrost, invoker settings
- https://docs.restate.dev/references/architecture — partitions, log, RocksDB
- https://docs.restate.dev/services/configuration — retry policy defaults, retention, timeouts
- https://docs.restate.dev/services/invocation/managing-invocations — cancel/kill/purge/pause/resume
- https://docs.restate.dev/services/flow-control.md — scopes, limit keys, rules
- https://docs.restate.dev/guides/rate-limiting.md — token bucket object
- https://docs.restate.dev/guides/error-handling — transient vs terminal, timeouts
- https://docs.restate.dev/guides/cron.md — cron pattern, overlap warning
- https://docs.restate.dev/develop/ts/durable-steps.md, https://docs.restate.dev/develop/ts/durable-timers.md — run warnings, no native cron
- https://docs.restate.dev/llms.txt — page index
- https://github.com/restatedev/restate/releases/tag/v1.5.0 (published 2025-09-16 per GitHub API) — retry policy pause default
- https://api.github.com/repos/restatedev/restate, /restatedev/sdk-rust — stars, release dates
- Failed (404): https://docs.restate.dev/develop/rust/{overview,journaling-results,error-handling,durable-timers,awakeables,workflows,service-communication,concurrent-tasks}; https://docs.restate.dev/operate/invocation (page exists but had no cancel/kill detail); https://docs.restate.dev/develop/rust.md redirects to docs.rs
- Not fetched verbatim (lead only): https://docs.restate.dev/references/errors — message_size_limit 32 MiB
