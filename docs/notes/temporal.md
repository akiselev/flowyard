# Temporal

Researched: 2026-09-16. Scope: docs.temporal.io (live pages, September 2026); temporalio/sdk-rust `main` and release v1.0.0 (published 2026-09-04, first non-prerelease; 520 stars, MIT); temporalio/temporal `main` (`schema/sqlite/v3/temporal/schema.sql`, `common/dynamicconfig/constants.go`); temporalio/sdk-go `internal/worker.go` for worker option comments.

## Summary
Temporal is a client/server durable-execution platform: a Temporal Service (frontend, history, matching, worker roles) persists an append-only Event History per Workflow Execution, and separate Worker processes poll task queues, replay history through user workflow code, and execute Activities. It optimizes for long-lived, distributed, multi-team orchestration with strong at-least-once activity semantics and exactly-once observation of results. The cost is a strict determinism contract on workflow code, a versioning ceremony for code changes, and an always-on server plus persistence store. The Rust SDK (`temporalio-sdk`) reached 1.0.0 on 2026-09-04 and is built on `sdk-core`, the same core used by the TypeScript, Python, .NET and Ruby SDKs.

## Deployment and process model
- Server/daemon: the Temporal Service is a separate long-running process cluster plus a database (Cassandra, MySQL, PostgreSQL; SQLite for the dev server). Nothing progresses unless the service is up AND at least one Worker is polling the relevant task queue.
- Workers are user processes: `Worker::new(&runtime, client, worker_options)?.run().await?` (Rust README). Workflow tasks and activity tasks are pulled: "Workers poll for Tasks in Task Queues via synchronous RPC" and "Workflow and Activity Tasks persist in a Task Queue. When a Worker Process goes down, the messages remain until the Worker recovers and can process the Tasks." (docs.temporal.io/task-queue)
- Single-node story: `temporal server start-dev`. "By default, Workflow Executions are lost when the server process dies." `--db-filename` = "Path to file for persistent Temporal state store." UI on 8233. "The development server is not intended for production use. It skips certain HTTP security checks to make local use simpler." (docs.temporal.io/cli/server)
- Verdict for Flowyard's question 3: Temporal assumes a long-running daemon (server) and long-running workers; a one-shot process can only act as a client/worker that connects to the running service.

## Execution model
History replay. Workflow code is re-executed from the start on every Workflow Task; the Commands it emits are matched against recorded Events.

Verbatim (docs.temporal.io/workflow-definition):
> "you must take care to ensure that any time your Workflow code is executed it makes the same Workflow API calls in the same sequence"

Commands that must not be reordered/added/removed without versioning (verbatim list):
- "Starting or cancelling a Timer"
- "Scheduling or cancelling Activity Executions (including local Activities)"
- "Starting or cancelling Child Workflow executions"
- "Signalling or cancelling signals to external Workflow Executions"
- "Scheduling or cancelling Nexus operations"
- "Ending the Workflow Execution in any way"
- "`Patched` or `GetVersion` calls for Versioning"
- "Upserting Workflow Search Attributes"
- "Upserting Workflow Memos"
- "Running a `SideEffect` or `MutableSideEffect`"

Safe changes without versioning: timer durations (SDK-specific exceptions), Activity/Child Workflow option arguments, signal parameters, adding handlers for previously unused signal types.

Replay check, verbatim: "When the Workflow's code replays, the Commands that are emitted are compared with the existing Event History. If a corresponding Event already exists within the Event History that matches that command, then the Execution progresses." ... "If a generated Command doesn't match what it needs to in the existing Event History, then the Workflow Execution returns a non-deterministic error".

Consequence: a non-determinism error is a Workflow Task Failure, which is transient: "These types of failures will cause the Workflow Task to be retried until the Workflow Execution Timeout, which is unlimited by default." (docs.temporal.io/references/failures). The workflow is stuck, not failed, until code is fixed or the run is reset/terminated.

Rust SDK determinism rules (crates/sdk/README.md): "Workflow code must be deterministic." No direct I/O, threading, RNG, system time; "do not use `tokio` or `futures` concurrency primitives directly in workflow code." The SDK ships `select!`, `join!`, `join_all` in `temporalio_sdk::workflows`. Internally: "Workflows run in a LocalSet ... The WorkflowExecutor replaces tokio::task::spawn_local for workflow tasks and provides custom wakers for nondeterminism detection." (crates/sdk/src/lib.rs). Side effects and local activities are recorded as `MarkerRecorded` events and "do not re-execute upon replay, but instead return the recorded result".

## Step and operation identity
- Identity is positional: each command gets a sequence number in the Workflow Task and is matched to the next event of the compatible type. Activities additionally carry an Activity Id: "The identifier for an Activity Execution. The identifier can be generated by the system, or it can be provided by the Workflow code" and "A single Workflow Run may reuse an Activity ID if an earlier Activity Execution with the same ID has closed" (docs.temporal.io/activity-execution). Activity Id is NOT used to look up results during replay; sequence is.
- Mismatch: non-deterministic error (above). sdk-core messages (crates/sdk-core/src/worker/workflow/machines/workflow_machines.rs): `"No command scheduled for event {event}"` (history has an event, replaying code produced no command for it), `"Non-deprecated patch marker encountered for change {patch_name}, but there is no corresponding change command!"`, `"During event handling, this event had an initial command ID but we could not find a matching command for it"`.
- Unreached recorded steps (Q1): if new code skips a recorded ActivityTaskScheduled/TimerStarted, the next event in history has no matching command and replay fails with the non-determinism error above; the run does not silently drop the orphan. The escape hatch is a `patched()` branch that keeps the old path for old runs.

## Persistence schema
Store: pluggable SQL/Cassandra. The SQLite v3 schema (github.com/temporalio/temporal/blob/main/schema/sqlite/v3/temporal/schema.sql) has, among others:
- `executions(shard_id, namespace_id, workflow_id, run_id, next_event_id, last_write_version, data MEDIUMBLOB, data_encoding, state MEDIUMBLOB, state_encoding, db_record_version)` — mutable state as a proto blob.
- `current_executions(shard_id, namespace_id, workflow_id, run_id, create_request_id, state INT, status INT, start_time, last_write_version, data, data_encoding)` PK `(shard_id, namespace_id, workflow_id)` — this row is what enforces one open run per Workflow Id.
- `history_node(shard_id, tree_id, branch_id, node_id, txn_id, prev_txn_id, data MEDIUMBLOB, data_encoding)` and `history_tree(...)` — batches of serialized events; branches support reset.
- `activity_info_maps(..., schedule_id, data, data_encoding)`, `timer_info_maps(..., timer_id, ...)`, `child_execution_info_maps(..., initiated_id, ...)`, `buffered_events`, `transfer_tasks`, `timer_tasks(shard_id, visibility_timestamp, task_id, data)`.
- No `schedules` table: Schedules are themselves workflows in the system namespace.
Step outputs live inline as payloads inside event blobs. Limits (constants.go, self-hosted defaults; Cloud page states the same numbers): `limit.blobSize.error` = 2*1024*1024 ("per event blob size limit"), warn at 512 KiB; `limit.historySize.error` = 50 MiB (warn 10 MiB); `limit.historyCount.error` = 51,200 events (warn 10,240). "The Workflow Execution is terminated when the Event History exceeds 51,200 Events" (docs.temporal.io/workflow-execution/event). gRPC message limit 4 MB. Large data: "Offload large payloads to an object store ... This pattern is built into the SDKs as External Storage, or you can implement your own claim check pattern by using a custom Payload Codec." (docs.temporal.io/troubleshooting/blob-size-limit-error). Codecs: "Encodes and decodes Payloads. Applies encryption, compression, or other byte-level transformations." running "In-process, inside your Workers and Clients" and a Codec Server for CLI/UI decoding (docs.temporal.io/codec-server).

## Crash recovery of in-flight work
- "The Temporal Server doesn't detect failures when a Worker loses communication with the Server or crashes. Therefore, the Temporal Server relies on the Start-To-Close Timeout to force Activity retries." (docs.temporal.io/encyclopedia/detecting-activity-failures). "We strongly recommend setting a Start-To-Close Timeout."
- Heartbeat Timeout: "the maximum time between Activity Heartbeats"; "0s means the Heartbeat Timeout is disabled." Heartbeat payloads act as progress checkpoints: "If an Activity Task Execution times out due to a missed Heartbeat, the next Activity Task can access and continue with that payload." Rust: `ctx.record_heartbeat(()).await?` (examples/cancellation/workflows.rs).
- Semantics: "For an Activity with a Retry Policy that allows retries, Temporal guarantees that the Activity will be observed as completed exactly once. However, the Activity may be executed multiple times." and "You should always make your business logic Activities idempotent" (docs.temporal.io/activity-definition). There is no per-activity "uncertain effect, stop for reconciliation" mode; the only knobs are Maximum Attempts = 1 or non-retryable error types.
- Workflow tasks: a worker crash mid-workflow-task is covered by the Workflow Task Timeout (10s default) after which the task is redelivered. Local activities are weaker: "A Local Activity result becomes durable only when the enclosing Workflow Task successfully completes and records a MarkerRecorded event."
- Timeouts vocabulary (verbatim): Schedule-To-Start "maximum amount of time that is allowed from when an Activity Task is scheduled ... to when a Worker starts that Activity Task", default infinity, "non-retryable by design"; Start-To-Close "the maximum time allowed for a single Activity Task Execution"; Schedule-To-Close "the maximum amount of time allowed for the overall Activity Execution".

## Retries, backoff, timeouts
Retry Policy fields and defaults (docs.temporal.io/encyclopedia/retry-policies, verbatim table):
```
Initial Interval     = 1 second
Backoff Coefficient  = 2.0
Maximum Interval     = 100 × Initial Interval
Maximum Attempts     = ∞
Non-Retryable Errors = []
```
- "Activities in Temporal are associated with a Retry Policy by default, while Workflows are not." Retries are per-activity (each attempt is a new Activity Task; attempt counter is in `ActivityTaskStarted`/activity info), or per-workflow when a workflow retry policy is set ("the failed run ends with `WorkflowExecutionFailed`, with `retryState=IN_PROGRESS`").
- Classification: "the Failure's type field is matched against a Retry Policy's list of non-retryable errors ... Activities and Workflow can also avoid retrying by setting an Application Failure's non_retryable flag to true." (docs.temporal.io/references/failures)
- Persistence of retry state: the server owns the retry timer (timer_tasks) and the attempt count; a worker restart does not reset them. Jitter: not mentioned on the retry-policy page (unverified whether the server adds jitter).
- Activity-level "cooldown requested by the server" (e.g. a 429 Retry-After) has no first-class field; the doc-recommended approach is heartbeating or failing with a retryable error.

## Flow control
Keyed by task queue and by worker, not by host/resource:
- `WorkerActivitiesPerSecond` (Go, sdk-go internal/worker.go): "rate limiting on number of activities that can be executed per second per worker ... This rate limit is applied after activity tasks are received by the worker. If this value is set very low, server-side timeouts may continue to elapse while tasks wait behind this worker-side rate limiter." Default 100k.
- `TaskQueueActivitiesPerSecond`: "This is managed by the server and controls activities per second for your entire taskqueue whereas WorkerActivityTasksPerSecond controls activities only per worker ... NOTE: Setting this to a non zero value will also disable eager activities." Default 100k. Rust equivalent in `WorkerOptions`: "Sets the maximum number of activities per second the task queue will dispatch" (crates/sdk/src/lib.rs).
- `MaxConcurrentActivityExecutionSize` default 1k per worker; newer "worker tuners"/slot suppliers supersede the maxConcurrent options (docs.temporal.io/develop/worker-performance).
- Cross-process: the task-queue rate limit is enforced by the matching service (cross-worker). Whether it survives a matching-service restart is unverified (it is a task-queue setting sent by workers on poll). Per-host cooldowns for scraping would have to be modelled as dedicated task queues per host, or as workflow-side logic.

## Fan-out, child workflows, map
- Parallel activities: `join_all` over `execute_activity` futures in the workflow (deterministic wrappers). The "work set" is not recorded up front as a unit; each scheduled activity is its own command/event, so a crash before the workflow task completes simply re-emits them on replay.
- Child Workflows: "A Child Workflow Execution is a Workflow Execution that is spawned from within another Workflow in the same Namespace." Each child has its own Workflow Id (Rust: `ChildWorkflowOptions::workflow_id(format!("greeting-child-{i}"))`, crates/sdk/examples/child_workflows/workflows.rs) and its own history. Parent Close Policy: Terminate (default, "the Child Workflow Execution is forcefully Terminated"), Abandon ("not affected"), Request Cancel. Guidance: "When in doubt, use an Activity."
- Continue-As-New: "allows you to checkpoint your Workflow's state and start a fresh Workflow" to escape history limits and versioning drift; "Temporal will tell your Workflow when it's approaching performance or scalability problems."
- Singleton per key (Q2): "Temporal guarantees that only one Workflow Execution with a given Workflow Id can be in an Open state at any given time." Workflow Id Reuse Policy (for closed runs): Allow Duplicate (default), Allow Duplicate Failed Only, Reject Duplicate, Terminate if Running. Workflow Id Conflict Policy (for open runs): Fail (default, "returns a `Workflow execution already started` error"), Use Existing ("returns a successful response with the Open Workflow Execution's Run Id"), Terminate Existing. (docs.temporal.io/workflow-execution/workflowid-runid)

## Versioning and code change policy
- Patching (Python/.NET/Ruby/TS; Rust exposes `SyncWorkflowContext::patched` plus a worker-level `patch_activation_callback`): "A Patch defines a logical branch in a Workflow for a specific change, similar to a feature flag." Lifecycle (docs.temporal.io/develop/python/versioning): (1) "Patch in any new, updated code using the `patched()` function. Run the new patched code alongside old code."; (2) "Remove old code and use `deprecate_patch()` to mark a particular patch as deprecated."; (3) "Once your pre-patch Workflows have left retention, you can then safely deploy Workers that no longer use either the `patched()` or `deprecate_patch()` calls." Rule: "always put the newest code at the top of an if-patched-block."
- Go `GetVersion(ctx, changeID, minSupported, maxSupported)` "records a marker in the Event History so that all future calls to `GetVersion` for this change Id ... will always return the given version number."
- Worker Versioning: "Each Deployment Version is identified by a deployment name and a Build ID." "A Pinned Workflow is guaranteed to complete on a single Worker Deployment Version." "An Auto-Upgrade Workflow will move to the latest Worker Deployment Version automatically whenever you change the current version." Old versions go Draining -> Drained. The pre-2025 experimental Worker Versioning "will be removed from Temporal Server in March 2026" (docs.temporal.io/develop/go/versioning).
- Replay testing: `crates/sdk/src/workflow_replayer.rs`; docs recommend replay tests to detect needed patches.

## Schedules, timers, missed runs
- Timers are server-side (`timer_tasks`), fired regardless of worker liveness; the workflow observes `TimerFired` on replay.
- Schedules (docs.temporal.io/schedule), overlap policy verbatim: Skip (default) "Nothing happens; the Workflow Execution is not started."; BufferOne "Starts the Workflow Execution as soon as the current one completes. The buffer is limited to one."; BufferAll "Allows an unlimited number of Workflows to buffer. They are started sequentially."; CancelOther; TerminateOther; AllowAll "Starts any number of concurrent Workflow Executions."
- Catchup Window: "The default is one year, meaning Actions will be taken unless over one year late." Minimum ten seconds. After downtime: "When it comes back up, the Catchup Window controls which missed Actions should be taken at that point." Backfill exists for manual replays. Pause-on-failure available.
- Rust client: `client.create_schedule(id, CreateScheduleOptions::builder().action(ScheduleAction::start_workflow(...)).spec(ScheduleSpec::from_interval(...)).trigger_immediately(true).build())`, `handle.trigger(ScheduleOverlapPolicy::Unspecified, ..)` (examples/schedules/starter.rs).

## Cancellation and late results
- Workflow cancel records `WorkflowExecutionCancelRequested`; "An Activity Execution is always canceled when its Workflow Execution is canceled." but "Activities must heartbeat to receive cancellations from a Temporal Service."
- Cleanup pattern: Go `workflow.NewDisconnectedContext`; Rust uses a fresh `WorkflowCancellationToken::new()` on the cleanup activity "to ensure cleanup activity is run" (examples/cancellation/workflows.rs). Go docs: "Even though the context is cancelled when the Workflow is Cancelled, you are still able to send Activity Heartbeats."
- Late results: activity completion after the cancel request is recorded as a normal completion/cancel of that activity execution; the workflow decides what to do. The docs do not spell out discard semantics (unverified beyond the above).

## Observability and tooling
- CLI `temporal workflow`: list, describe, show ("Show a Workflow Execution's Event History"), start/execute, signal, query, update, cancel, terminate, reset (types FirstWorkflowTask, LastWorkflowTask, LastContinuedAsNew), trace (real-time progress incl. children), stack, pause/unpause (experimental), delete, fix-history-json. Web UI bundled with dev server.
- Reset = create a new run from a history point; combined with replay testing this is the "retry from step" story. Histories are exportable JSON.

## API ergonomics
Minimal complete example, verbatim from github.com/temporalio/sdk-rust/blob/main/crates/sdk/README.md:
```rust
use temporalio_macros::activities;
use temporalio_sdk::activities::{ActivityContext, ActivityError};
use std::sync::{Arc, atomic::{AtomicUsize, Ordering}};

struct MyActivities {
    counter: AtomicUsize,
}

#[activities]
impl MyActivities {
    #[activity]
    pub async fn greet(_ctx: ActivityContext, name: String) -> Result<String, ActivityError> {
        Ok(format!("Hello, {}!", name))
    }

    #[activity]
    pub async fn increment(self: Arc<Self>, _ctx: ActivityContext) -> Result<u32, ActivityError> {
        Ok(self.counter.fetch_add(1, Ordering::Relaxed) as u32)
    }
}
```
```rust
use temporalio_macros::{workflow, workflow_methods};
use temporalio_sdk::{WorkflowContext, WorkflowContextView, WorkflowResult};
use std::time::Duration;

#[workflow]
pub struct GreetingWorkflow {
    name: String,
}

#[workflow_methods]
impl GreetingWorkflow {
    #[init]
    fn new(_ctx: &WorkflowContextView, name: String) -> Self {
        Self { name }
    }

    #[run]
    async fn run(ctx: &mut WorkflowContext<Self>) -> WorkflowResult<String> {
        let name = ctx.state(|s| s.name.clone());

        let greeting = ctx.execute_activity(
            MyActivities::greet,
            name,
            ActivityOptions::start_to_close_timeout(Duration::from_secs(10))
        )?.await?;

        Ok(greeting)
    }
}
```
```rust
use temporalio_client::{Client, ClientOptions, Connection, envconfig::LoadClientConfigProfileOptions};
use temporalio_sdk::{Runtime, Worker, WorkerOptions};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let runtime = Runtime::from_current_tokio(Default::default())?;
    let (conn_options, client_options) = ClientOptions::load_from_config(
        LoadClientConfigProfileOptions::default()
    )?;
    let connection = Connection::connect(conn_options).await?;
    let client = Client::new(connection, client_options)?;

    let worker_options = WorkerOptions::new("my-task-queue")
        .register_activities(MyActivities { counter: Default::default() })
        .register_workflow::<GreetingWorkflow>()?
        .build();

    Worker::new(&runtime, client, worker_options)?.run().await?;
    Ok(())
}
```
Delightful:
- Activities referenced as typed function paths (`MyActivities::greet`) with typed inputs/outputs; `#[activities]` on an impl block gives shared state via `Arc<Self>`.
- `&mut WorkflowContext<Self>` for the run method (the same "borrow mutably" shape Flowyard proposes); `ActivityContext` is a separate type with `record_heartbeat`, `is_cancelled`.
- Deterministic `select!`/`join!`/`join_all` wrappers, a replayer for tests, and a single vocabulary of timeouts that maps directly to failure modes.
- Schedules and child workflows are first-class client/SDK types, not user plumbing.
Painful / footguns:
- The determinism list above is enforced only at replay time; a mistake shows up as a stuck workflow that "will cause the Workflow Task to be retried until the Workflow Execution Timeout, which is unlimited by default."
- Every code change that touches command order needs `patched`/`GetVersion` with a three-deploy lifecycle, or Worker Versioning with drain management.
- 2 MB per payload, 4 MB per gRPC message, 50 MB / 51,200 events per history; a scraper that stores page bodies inline hits these quickly and must adopt External Storage/claim-check.
- Local activities: "do not support Activity heartbeats", and "Signals and other external Workflow events are not processed until the Local Activities finish."
- Rust SDK is 12 days past 1.0 at research time; docs.rs marks plugins, Nexus and versioning modules experimental; there is no Rust page on docs.temporal.io, only repo examples.
- Requires a running server; dev server is explicitly non-production.
Scraping pipeline sketch: one `RefreshSource` workflow per source with Workflow Id = source id (conflict policy Fail gives the singleton); `fetch_page` activity with Start-To-Close + heartbeat and a retry policy with `Non-Retryable Errors = ["ParseSchemaChanged"]`; `join_all` over per-page fetch activities (or a child workflow per page batch when a page set exceeds a few hundred events); results too big for 2 MB go to an object store via a codec/External Storage and only references flow through history; `publish_snapshot` as a final activity; a Schedule with `Skip` overlap and a short catchup window. User-written plumbing: payload offloading, per-host rate limits (task queue per host or in-activity limiter), idempotency keys for publish, patch branches on every reorder.

## Known limitations
- History limits terminate the run (51,200 events / 50 MB) — docs.temporal.io/workflow-execution/event, constants.go.
- Per-event blob limit 2 MB, gRPC 4 MB — docs.temporal.io/troubleshooting/blob-size-limit-error.
- Server "doesn't detect failures when a Worker loses communication"; correctness depends on timeouts — detecting-activity-failures page.
- Non-determinism errors block rather than fail (references/failures).
- Local activities: no heartbeat, block signal processing (docs.temporal.io/local-activity).
- Dev server: in-memory by default, "not intended for production use" (cli/server).
- Pre-2025 Worker Versioning removed March 2026 (develop/go/versioning).
- Rust SDK: experimental modules (docs.rs temporalio-sdk 1.0.0); `register_workflow_with_factory` "can easily cause nondeterminism" (lib.rs).

## Relevance to Flowyard
- Borrow:
  - The timeout vocabulary (schedule-to-start / start-to-close / schedule-to-close / heartbeat) and the rule that the engine "relies on the Start-To-Close Timeout" rather than detecting crashes — Flowyard's single-owner model can replace timeouts with process-exit detection but should still expose a per-activity deadline.
  - Heartbeat details as a resumable progress checkpoint for long fetches.
  - Retry policy shape (initial, coefficient, max interval, max attempts, non-retryable error types) and `non_retryable` flag on application errors.
  - Workflow Id Conflict Policy semantics (Fail / Use Existing / Terminate Existing) as the singleton-per-key API.
  - Schedule overlap policies and the explicit Catchup Window as the missed-run policy.
  - A replayer test harness and `reset`-to-point semantics for "retry from step".
  - Disconnected cancellation scope for cleanup activities.
- Avoid:
  - Positional command identity; Flowyard's keyed operations are meant to avoid exactly the "No command scheduled for event" class of failure.
  - The patch/deprecate/remove ceremony; a build-digest gate with old-binary completion is simpler for finite refresh runs.
  - Inline payload limits designed for a shared cluster; Flowyard should route large outputs to Artifactum by design.
  - Server + worker + database topology.
- Answers to open questions:
  - Q1: unreached recorded commands make replay fail with a non-deterministic error; the workflow task is retried indefinitely until code is patched or the run is reset (workflow-definition; references/failures; sdk-core `"No command scheduled for event"`).
  - Q2: yes; "only one Workflow Execution with a given Workflow Id can be in an Open state"; expressed via Workflow Id + Conflict/Reuse policies (workflowid-runid).
  - Q3: long-running server and workers; no one-shot mode (task-queue, cli/server).
  - Q4: dispatched via persisted task queues that workers long-poll; local activities are the inline exception with weaker durability (task-queue; local-activity).
  - Q5: per-worker (in-memory) and per-task-queue (server-managed) activities-per-second; keyed by task queue, not by host (sdk-go worker.go).
  - Q6: local activities for "short-lived (completes in a few seconds), including retries", idempotent, no global rate limit needs; everything else a regular activity; pure computation stays in workflow code if deterministic (local-activity).
  - Q7: inline in event blobs, 2 MB per payload; External Storage/claim-check replaces payloads with references (blob-size-limit-error; codec-server).
  - Q8: explicit version markers (`patched`/`GetVersion`) recorded in history, or Build-ID pinning; no digest gate, no warn-and-continue (versioning pages).
  - Q9: server-owned attempt counter and retry timers; policy declared per activity; jitter unverified (retry-policies; schema timer_tasks).
  - Q10: Catchup Window (default 1 year) plus overlap policy; missed actions within the window are executed, Skip/BufferOne bound the burst (schedule).

## Sources
- https://docs.temporal.io/workflow-definition — determinism constraints, replay mismatch
- https://docs.temporal.io/workflow-execution/event — history limits, side effects
- https://docs.temporal.io/encyclopedia/retry-policies — retry fields/defaults
- https://docs.temporal.io/encyclopedia/detecting-activity-failures — timeouts, heartbeats
- https://docs.temporal.io/workflow-execution/workflowid-runid — ID reuse/conflict policies
- https://docs.temporal.io/patching — patched()
- https://docs.temporal.io/develop/python/versioning — patch lifecycle
- https://docs.temporal.io/develop/go/versioning — GetVersion, worker versioning removal date
- https://docs.temporal.io/worker-versioning — Pinned/AutoUpgrade, draining
- https://docs.temporal.io/schedule — overlap policy, catchup window
- https://docs.temporal.io/workflow-execution/continue-as-new
- https://docs.temporal.io/child-workflows
- https://docs.temporal.io/parent-close-policy
- https://docs.temporal.io/cloud/limits — 2 MB / 4 MB / 50 MB / 51,200
- https://docs.temporal.io/troubleshooting/blob-size-limit-error — claim check / External Storage
- https://docs.temporal.io/codec-server
- https://docs.temporal.io/dataconversion (little detail on this page)
- https://docs.temporal.io/local-activity
- https://docs.temporal.io/activity-execution — cancellation, Activity Id
- https://docs.temporal.io/activity-definition — idempotency, at-least-once
- https://docs.temporal.io/references/failures — workflow task failure retry
- https://docs.temporal.io/develop/go/cancellation, https://docs.temporal.io/develop/python/cancellation
- https://docs.temporal.io/develop/worker-performance (tuners; legacy options not described there)
- https://docs.temporal.io/task-queue
- https://docs.temporal.io/cli/server, https://docs.temporal.io/cli/workflow
- https://github.com/temporalio/sdk-rust and https://raw.githubusercontent.com/temporalio/sdk-rust/main/crates/sdk/README.md — Rust example
- https://raw.githubusercontent.com/temporalio/sdk-rust/main/crates/sdk/src/lib.rs — nondeterminism detection comment, patch callback, task-queue rate option
- https://raw.githubusercontent.com/temporalio/sdk-rust/main/crates/sdk-core/src/worker/workflow/machines/workflow_machines.rs — nondeterminism messages
- https://raw.githubusercontent.com/temporalio/sdk-rust/main/crates/sdk/examples/{child_workflows/workflows.rs,schedules/starter.rs,cancellation/workflows.rs,activity_heartbeating/README.md}
- https://docs.rs/temporalio-sdk/latest/temporalio_sdk/ — 1.0.0, experimental modules
- https://api.github.com/repos/temporalio/sdk-rust/releases — v1.0.0 published 2026-09-04
- https://raw.githubusercontent.com/temporalio/temporal/main/schema/sqlite/v3/temporal/schema.sql
- https://raw.githubusercontent.com/temporalio/temporal/main/common/dynamicconfig/constants.go — limit defaults
- https://raw.githubusercontent.com/temporalio/sdk-go/master/internal/worker.go — rate-limit option comments
- Tried, not useful: https://pkg.go.dev/go.temporal.io/sdk/worker#Options (field docs not rendered in fetch)
