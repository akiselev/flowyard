# Obelisk

Researched: 2026-09-16. Scope: obeli.sk docs "latest" (v0.41.5, released 2026-08-31); github.com/obeli-sk/obelisk `main` (README, ROADMAP.md, `crates/db-sqlite/migrations/V1..V21`, `assets/schemas/toml/authored.json`, `crates/wasm-workers/src/workflow/event_history.rs`, `crates/wasm-workers/src/cron/cron_worker.rs`, `wit/obelisk_workflow@7.0.0`, `wit/obelisk_types@6.0.0`); obelisk-templates fibo README. Repo: 755 stars, AGPL-3.0 (wit/ and proto/ MIT), created 2024-04-25, single maintainer org.

## Summary
Obelisk is "a deterministic workflow engine built on the WASM Component Model": a single Rust binary that runs workflows, activities and webhook endpoints compiled to WASM (Rust or JavaScript), persisting an append-only execution log in SQLite or PostgreSQL. Workflows are sandboxed (compiled for `wasm32-unknown-unknown`, no WASI), so the only sources of non-determinism are host functions the runtime records; activities run under WASI 0.2 with HTTP and must be idempotent. It optimizes for auditability and crash-safe replay of small-to-medium orchestration (periodic tasks, batch jobs, AI-agent sandboxing) with a strong type contract (WIT). Status: "**Pre-release**: Expect changes in CLI, gRPC, WIT, and database schema." (README).

## Deployment and process model
- Single daemon: `obelisk server run --deployment deployment.toml [--server-config server.toml]`. Ports: Web UI 8080, API 5005 (bearer token required since 0.40.0), user HTTP servers for webhooks (e.g. 9090). "Single binary, embedded SQLite or Postgres. No brokers, no sidecars, no YAML pipelines." (obeli.sk)
- Work-stealing executors inside the process: "Work Stealing Executor — Concurrency limits and customizable retry handling." Per-component `exec.batch_size` (5), `exec.lock_expiry` (30 s default in schema; tutorial uses 1–10 s), `exec.tick_sleep` (200 ms), `exec.instance_limiter`, global `wasm.global_executor_instance_limiter`.
- A Deployment is "a versioned snapshot of your component configuration"; `t_deployment` has unique partial indexes enforcing "at most one active deployment at a time" and one enqueued. CLI: `deployment submit|enqueue|apply|list|active|show|get|gc`.
- Q3: daemon. Nothing progresses when the server is down; overdue executions (`pending_at` in the past) are picked up on restart. There is no library mode.

## Execution model
- Replay from the execution log: "Every call, sleep, and result is persisted to the execution log." "On crashes, workflows restart and replay completed steps from the execution log." (obeli.sk). "The runtime records all non-deterministic calls — random values, timestamps, child results — to the execution log."
- Determinism is enforced by the sandbox rather than by rules: workflows compile to `wasm32-unknown-unknown` (no WASI clock/random/network); the only imports are `obelisk:workflow/workflow-support` (`random-u64`, `random-string`, `sleep`, join-set functions, `execution-id-generate`, `stub-json`), `obelisk:log`, and the generated per-activity extension interfaces. Docs: "Workflows cannot read environment variables or ambient host state. Instead, supply runtime configuration as workflow parameters." JS runtime: "Math.random() and Date.now() are safe in workflow code. Values are recorded on first call and replayed identically on crash recovery."
- Event log: "Each event is assigned a **version** number equal to its index in the log (starting at 0 for the `Created` event)"; the version is "an optimistic concurrency control token: when an executor appends events, it must specify the expected current version." Event kinds: Created, Locked, Unlocked, Finished, Paused, Unpaused, CancellationRequested, ComponentUpgradeFinished, TemporarilyFailed, TemporarilyTimedOut; history events (while Locked): Persist, JoinSetCreate, JoinSetRequest, JoinNext, JoinNextTry, JoinNextTooMany, Schedule, Stub; responses: ChildExecutionFinished, DelayFinished.
- Blocking strategy per workflow: `await` "keeps instance waiting up to lock expiry"; `interrupt` "unloads instance immediately; replays execution log upon result arrival".

## Step and operation identity
- Positional with structural keys. During replay "the runtime compares the workflow's requests against the recorded history events to detect nondeterminism" (execution-log). Source (event_history.rs, `ApplyError::NondeterminismDetected`): `"key does not match event stored at version {version}: key: {key}, event: {found}"`; `"JoinNextTry recorded outcome=Found but no response available for join set ..."`; and on finishing: `"found unprocessed request stored at version {version}: event: {first_unprocessed}"`.
- Identifiers (concepts/identifiers): top-level `E_<ULID>`; child = parent id + join set id + counter, e.g. `E_01K42AJA8QHD7D3GTJAYMASFY.n:fetch_1`; join sets `o:N` (one-off), `g:N` (generated), `n:NAME` (named; charset alphanumeric plus `-/`); delays `Delay_<parent>.<joinset>_N`. Named join sets give the author a domain-keyed handle, but children within a set are still counter-indexed by submission order.
- Q1 (unreached recorded steps): the log check on completion fails the replay with `NondeterminismDetected("found unprocessed request stored at version ...")`. Failure kind `nondeterminism-detected` is a platform failure (`execution-failure-kind` in types.wit). With the default `auto` locking strategy the runtime replays in-progress executions against the new component after redeploy and "If replay detects issues, the execution stays on its previous component version" (`t_state.incompatible_digest`, migration V3); such executions need the old component (`obelisk execution upgrade`, ROADMAP: "Ability to drive old workflows to completion when auto upgrade fails - dormant deployments").

## Persistence schema
SQLite (and mirrored Postgres migrations), STRICT tables; verbatim from `crates/db-sqlite/migrations/V1__initial.sql` (trimmed to columns):
```sql
-- Stores execution history. Append only.
CREATE TABLE IF NOT EXISTS t_execution_log (
    execution_id TEXT NOT NULL, created_at TEXT NOT NULL, json_value TEXT NOT NULL,
    version INTEGER NOT NULL, variant TEXT NOT NULL, join_set_id TEXT,
    history_event_type TEXT GENERATED ALWAYS AS (json_value->>'$.history_event.event.type') STORED,
    PRIMARY KEY (execution_id, version)) STRICT;
-- Stores child execution return values for the parent (`execution_id`). Append only.
CREATE TABLE IF NOT EXISTS t_join_set_response (
    id INTEGER PRIMARY KEY AUTOINCREMENT, created_at TEXT NOT NULL, execution_id TEXT NOT NULL,
    join_set_id TEXT NOT NULL, delay_id TEXT, delay_success INTEGER,
    child_execution_id TEXT, finished_version INTEGER,
    UNIQUE (execution_id, join_set_id, delay_id, child_execution_id)) STRICT;
-- Stores executions in `PendingState`
CREATE TABLE IF NOT EXISTS t_state (
    execution_id TEXT NOT NULL, is_top_level INTEGER NOT NULL, corresponding_version INTEGER NOT NULL,
    ffqn TEXT NOT NULL, created_at TEXT NOT NULL, component_id_input_digest BLOB NOT NULL,
    component_type TEXT NOT NULL, first_scheduled_at TEXT NOT NULL, deployment_id TEXT NOT NULL,
    is_paused INTEGER NOT NULL, pending_expires_finished TEXT NOT NULL, state TEXT NOT NULL,
    updated_at TEXT NOT NULL, intermittent_event_count INTEGER NOT NULL,
    max_retries INTEGER, retry_exp_backoff_millis INTEGER, last_lock_version INTEGER,
    executor_id TEXT, run_id TEXT, join_set_id TEXT, join_set_closing INTEGER, result_kind TEXT,
    PRIMARY KEY (execution_id)) STRICT;
-- Represents `ExpiredTimer::AsyncDelay`. Rows are deleted when the delay is processed.
CREATE TABLE IF NOT EXISTS t_delay (execution_id TEXT NOT NULL, join_set_id TEXT NOT NULL,
    delay_id TEXT NOT NULL, expires_at TEXT NOT NULL, is_paused INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (execution_id, join_set_id, delay_id)) STRICT;
```
Plus `t_log` (structured logs/stdout per run), `t_execution_backtrace`/`t_wasm_backtrace`/`t_source_file`/`t_component_source` (time-travel debugger), `t_deployment`, and later `t_system_event` (V21). Indexes encode the scheduler: `WHERE state = 'pending_at'` on `(pending_expires_finished, ffqn)`, `WHERE state = 'locked'` on `pending_expires_finished` (expired-lock sweep). Later migrations: V3 `incompatible_digest`, V12 `lifecycle IN ('active','paused','cancelling')` replacing `is_paused`, V13 per-execution `seq` for responses, V18 `tombstoned`.
- Step outputs: inline as JSON text in `json_value` (params, results, persisted values). Limit: "The server defaults to a 1 MiB compact JSON-encoded limit for each newly persisted execution parameter array, result, stub result, persisted history value, or failure diagnostic" (`[limits] max_persisted_value_size_bytes`); "Each top-level execution snapshots the effective limit. Child and scheduled executions inherit it" (README). Exceeding it is `ApplyError::PersistedValueTooLarge` / failure kind `value-too-large`. No built-in blob/reference store.
- SQLite pragmas from config docs: `database.sqlite.pragma = { "cache_size" = "10000", "synchronous" = "FULL" }`; Postgres with `provision_policy`.

## Crash recovery of in-flight work
- Lease-based, even on one node: executors `Locked` an execution with `lock_expires_at` (= `pending_expires_finished` while `state='locked'`). "If the executor crashes without appending any event, other executors can acquire the execution after the lock expires." Transition Locked -> PendingAt (execution-states). Workflows extend locks (`lock_extension = true`, `lock_extension_leeway` 15 s default: "Starts extending the lock shortly before it expires").
- Activities: "Activities must be safe to execute more than once with the same logical input. Repeating the activity must not produce an incorrect duplicate side effect. This is the most important activity contract." "Automatic retries on errors, timeouts, and panics (WASM traps)." A crashed attempt with no outcome becomes `TemporarilyTimedOut` after lock expiry and is retried within `max_retries`.
- Workflows: "Automatic retries on timeouts"; a workflow crash mid-run replays from the log on the next lock.
- No user-declared "idempotent vs uncertain" policy; the contract is idempotency, period. A partially-applied external effect is retried.

## Retries, backoff, timeouts
- Declared per component in `deployment.toml`: activities `max_retries = 5`, `retry_exp_backoff.milliseconds = 100` (defaults from authored.json); workflows only `retry_exp_backoff` (no `max_retries` field — retries on timeouts are unbounded, unverified beyond schema absence); `exec.lock_expiry` is the per-attempt timeout.
- Persisted: `t_state.max_retries`, `retry_exp_backoff_millis`, `intermittent_event_count` (attempt counter), and the absolute next-eligible time in `pending_expires_finished` when `state='pending_at'`; `TemporarilyFailed`/`TemporarilyTimedOut` events in the log. Jitter: none seen in schema/docs (unverified).
- Classification (types.wit `execution-failure-kind`): `timed-out`, `nondeterminism-detected`, `out-of-fuel`, `cancelled`, `value-too-large`, `uncategorized`; business errors are the `err` arm of the WIT `result` and are returned to the parent, not retried. Workflow functions must return `result`, `result<T>`, `result<T, string>` or `result<T, E>` where E has an `execution-failed` variant.
- Fuel (`wasm.fuel`, default unlimited) bounds CPU per instance.

## Flow control
- Per-component `exec.instance_limiter` and global `global_executor_instance_limiter` / `global_webhook_instance_limiter` (defaults "unlimited"); `exec.batch_size` per tick; `[[http_server]] max_inflight_requests`. All in-memory limits of the single process, configured by component, not by host or key.
- Fairness knobs (0.41.1): `max_events_per_run = 100`, `max_replay_captured_writes = 100`, `response_refresh_interval = 32`.
- No rate limiting, cooldowns or keyed throttles. ROADMAP "Future ideas": "Queue capacity setting, adding backpressure to execution submission"; "Optional caching of activity executions with a TTL - serve cached response if parameters are the same".
- Structured concurrency provides backpressure at the parent: "the workflow will be marked as finished only when all unattended child executions are finished".

## Fan-out, child workflows, map
- Join sets (concepts/workflows/join-sets): `join_set_create()` / `join_set_create_named("some-name")`; `-submit` "persists the execution request to the database, queuing it for processing"; `-await-next` "returns results based on completion order, not submission order"; `-get` retrieves a specific execution's result after it was awaited; `-invoke` = labelled synchronous call; `-schedule` for fire-and-forget/deferred top-level executions; `submit-delay`/`join-next-try` for timeouts and polling. "Operations in a join set are completed in an arbitrary order ... The `await-next` call returns results in the order they become available. This does not break the determinism, as responses maintain their own ordering." (responses get a per-execution `seq`, migration V13).
- Work set recorded up front: every `-submit` is a `JoinSetRequest` event before the child exists as `t_state`; child ids are derived deterministically from parent + join set + counter.
- Close semantics (WIT `join-set-close`): "Unawaited delay requests, activities and cancellable workflows are cancelled, unawaited non-cancellable workflows are awaited, as mandated by structured concurrency pattern." "child executions cannot outlive their parent".
- Failure policy: each child result is a WIT `result`; the parent decides. Platform failures are surfaced via `get-execution-failure-kind`.
- Q2 singleton per key: not built in. The `patterns/generation-reconciler` doc implements it in user space: "keep one current workflow for each logical key, cancel obsolete or duplicate workflows, and start missing ones"; races are tolerated ("the next run keeps the newest current workflow and cancels the duplicate"). Top-level execution ids can be chosen by the submitter (`execution-id-generate`, `schedule-json` with a supplied id), which gives idempotent submission by id (uniqueness enforced by `t_state` primary key), but not "one active per key".

## Versioning and code change policy
- Deployments carry a digest and the stored `deployment.toml` + component files (content-addressed `t_file`, migrations V7–V9, V15). `t_state.component_id_input_digest` pins each execution to a component.
- Locking strategy `auto` (default for workflows): "in-progress executions are replayed against the current workflow component after redeployment. Compatible executions are upgraded and associated with the current deployment." ... "If replay detects issues, the execution stays on its previous component version." `by_component_digest`: "executions remain pending until explicitly upgraded" (`obelisk execution upgrade <id>`). `ComponentUpgradeFinished` event records the switch.
- So: warn-and-continue for compatible runs (verified by actual replay, not by declaration), stall for incompatible ones; no version attribute in workflow code.
- Schema versioning: numbered SQL migrations (V1..V21) applied on startup; the pre-release notice covers "database schema". 0.40 -> 0.41 migration notes list breaking config/API changes (secrets and outbound HTTP allowlists in `server.toml`, `execution_failed` spelling, gRPC `CancelExecution`).

## Schedules, timers, missed runs
- Persistent sleep: `sleep(schedule-at, name)` "Block execution for given time, return the time when the durable sleep expires." `schedule-at` = `now | at(datetime) | in(duration)`. Delay rows live in `t_delay(expires_at)` and wake via the sweep index (V14). "obelisk.sleep() is durable: its position is saved to the execution log."
- Cron: `[[cron]] name/ffqn/schedule/params` in `deployment.toml`; `@daily`, `@hourly`, ..., `@once` ("submits the target function once immediately and then the scheduler execution finishes"). "the scheduler is itself a durable execution, it survives server restarts and crashes without missing ticks"; across redeploys "the existing seed execution is reused with its full event history, so the next tick is computed from where it left off."
- Missed runs after downtime (Q10): not documented. Source (cron_worker.rs): on each tick the worker creates one scheduled execution with `scheduled_at: now`, then `next_fire_time(now)` and appends `Unlocked { unlocked_at: next_fire }`. Inference: a tick that became due during downtime runs once when the server returns, and the next tick is computed from the current time, so a week of downtime yields one catch-up run per cron, not a burst. Overlap is not addressed (each tick is an independent top-level execution).

## Cancellation and late results
- States: `Cancelling` "Cancellation has been requested. An activity is being stopped, or a cancellable workflow is closing its join sets." `obelisk execution cancel <id>` "For activities, cancellable workflows, or delays". Workflows opt in by exporting a function whose name ends in `-cancellable`; cancellation "closes their own join sets from the persisted execution log, recursively applying the same rules" (pending activities cancelled, running ones interrupted, non-cancellable child workflows awaited).
- Late results: a child result arriving after its parent's join set was closed is recorded as a response (append-only `t_join_set_response`, unique per child id) but the parent already processed the close; the docs do not discuss it further (unverified).
- Compensation: the "Cleanup Supervisor (Saga)" pattern nests workflows to guarantee cleanup after crashes.

## Observability and tooling
- CLI (docs/latest/cli): `server run|verify`, `deployment submit|enqueue|apply|list|active|show|get|gc`, `execution submit [--follow|--paused|--json]|list|status|result|logs|events|responses|cancel|stub|pause|unpause|replay ("Checking for non-determinism")|advance [--trim --pause-all --force]|upgrade`, `component list|push|add`, `generate wit-extensions|wit-support|wit-deps|server-config|deployment|deployment-id|execution-id|token|prompt`.
- Web UI: "a time-traveling debugger showing backtraces and sources of recorded events"; execution logs, HTTP traces; "Replay & Advance: Users can pause executions, step through events, and preview or apply writes". Structured logs and stdout/stderr per run in `t_log` (`forward_stdout = "db"`). gRPC + REST APIs with OpenAPI schema; `t_system_event` for server-level events.

## API ergonomics
Minimal example, verbatim from obeli.sk/docs/latest/wasm/getting-started/ (WIT, activity, deployment):
```wit
package tutorial:activity;

interface activity-sleepy {
    step: func(idx: u64, sleep-millis: u64) -> result<u64>;
}
```
```rust
impl Guest for Component {
    fn step(idx: u64, sleep_millis: u64) -> Result<u64, ()> {
        println!("Step {idx} started");
        std::thread::sleep(std::time::Duration::from_millis(sleep_millis));
        println!("Step {idx} completed");
        Ok(idx)
    }
}
```
Workflow fan-out through generated extension bindings (wasm/rust-components):
```rust
step_submit(&join_set, i, i * 200);  // non-blocking submit
let result = step_await_next(&join_set).unwrap();
```
```toml
[[activity_wasm]]
name = "activity_sleepy"
location = "target/wasm32-wasip2/release/activity_sleepy.wasm"
exec.lock_expiry.seconds = 10

[[workflow_wasm]]
name = "workflow_tutorial"
location = "target/wasm32-unknown-unknown/workflow/workflow_tutorial.wasm"
```
Build: `cargo build --release --target wasm32-wasip2` (activity), `cargo build --profile workflow --target wasm32-unknown-unknown` (workflow); `cargo generate obeli-sk/obelisk-templates`; crates need `crate-type = ["cdylib"]` and `wit-bindgen`'s `generate!({ generate_all })` plus `export!(Component)`. "Obelisk automatically converts the Core WASM Module to a component during server startup."
Delightful:
- Determinism you cannot violate by accident: the workflow target has no clock, RNG or network; recorded host functions are the only escape.
- Schema-first, typed contracts (WIT) between workflow and activities; generated `-submit/-await-next/-get/-schedule/-invoke` per function; JSON encoding for CLI/UI submission.
- Structured concurrency with durable membership; "Unawaited activities auto-cancel on parent completion."
- Single binary + SQLite with `synchronous = FULL`, full CLI, time-travel debugger, `execution replay` for non-determinism checks, cron in config.
Painful / footguns:
- Two WASM targets, custom cargo profiles, `wit/deps` management and code generation before you write a line of orchestration; workflows are synchronous WIT functions, not async Rust, so ordinary Rust libraries (tokio, reqwest, headless browsers) are unavailable in workflows and only WASI-0.2-compatible crates work in activities.
- Everything persisted is JSON text in SQLite with a 1 MiB default per value and no reference type; large page bodies must be pushed to external storage by the activity.
- Nondeterminism leaves executions stuck on the old component until an operator upgrades them; pre-release schema and WIT major bumps (0.40 -> 0.41 renamed error variants and gRPC calls).
- No rate limiting, no per-key singleton, no retry jitter; retries are per component, not per call site.
- AGPL-3.0 for the runtime.
Scraping pipeline sketch: WIT `refresh-source: func(source: source-spec) -> result<snapshot-ref, string>` workflow; activities `fetch-page(url) -> result<page-ref>` (WASI HTTP, writes body to object storage and returns a key), `parse(page-ref) -> result<list<event>>`, `enrich(event) -> result<event>`, `publish(snapshot) -> result<snapshot-ref>`; the workflow creates a named join set `n:fetch`, `fetch_page_submit` for each URL, `fetch_page_await_next` in a loop, then a second join set for enrich, then `publish`. Per-host cooldown: not available; emulate by serializing per-host fetches in the workflow or by an activity-side limiter. Singleton per source: a reconciler cron workflow per the generation-reconciler pattern. Schedule: `[[cron]]` entry per source or one cron that fans out.

## Known limitations
- "Pre-release: Expect changes in CLI, gRPC, WIT, and database schema." (README)
- 1 MiB persisted-value default, JSON-inline storage (README `Persisted value limit`).
- No native rate limiting/backpressure on submission (ROADMAP future ideas).
- Failed auto-upgrade leaves executions on the old component (deployments page; ROADMAP "dormant deployments").
- Cron missed-run and overlap semantics undocumented (concepts/cron).
- Workflows limited to WIT-expressible types and JSON encoding; activities limited to WASI 0.2 (README, runtime-support).
- Exec activities disabled by default since 0.40.0; API requires bearer token (releases).

## Relevance to Flowyard
- Borrow:
  - Join sets as the durable-membership fan-out primitive: request persisted before the child exists, deterministic child ids (`parent.joinset_N`), completion-order awaiting that stays deterministic because responses carry their own sequence.
  - Structured teardown rules on close/cancel (cancel activities and delays, await non-cancellable children) and "child executions cannot outlive their parent".
  - The state table shape: `t_state` with `pending_expires_finished` doing triple duty (next-eligible time, lock expiry, finish time) plus `intermittent_event_count`, `max_retries`, `retry_exp_backoff_millis` persisted per execution; append-only `t_execution_log` keyed `(execution_id, version)` with optimistic version tokens.
  - `execution-failure-kind` as the platform-failure taxonomy separate from business `err`.
  - Per-execution snapshot of size limits so config changes cannot alter replay.
  - Replay-verified compatibility (`auto` upgrade) as a better-than-declaration gate, and `execution replay` as a test command.
  - Deployment as a content-addressed, immutable snapshot with one active at a time.
- Avoid:
  - WASM component toolchain as the price of determinism; Flowyard wants plain async Rust.
  - JSON-inline outputs with a hard size cap and no reference type.
  - Counter-based child identity within join sets when domain keys exist.
  - Leaving incompatible runs silently parked on an old component; Flowyard should fail loudly with the build identity.
- Answers to open questions:
  - Q1: replay fails with `NondeterminismDetected("found unprocessed request stored at version ...")`; the execution stays pinned to the old component digest (event_history.rs; deployments page; migration V3).
  - Q2: no built-in singleton per key; user-space generation reconciler pattern (patterns/generation-reconciler).
  - Q3: long-running daemon; overdue `pending_at` work runs on restart (execution-states; t_state indexes).
  - Q4: dispatched: `-submit` persists a request, executors pull `pending_at` rows in batches under a lock; nothing executes inline in the workflow instance (join-sets page; t_state; exec.batch_size).
  - Q5: in-memory per-component instance limiters and global limiters; no host/key rate limits (configuration page; ROADMAP).
  - Q6: pure computation inside the sandbox is free; every host call (random, sleep, child) is a persisted event; there is no notion of a cheap unrecorded side effect because side effects are impossible in workflows (workflow-support WIT; features page).
  - Q7: inline JSON in `t_execution_log`/responses, 1 MiB default per value, snapshotted per execution tree (README; ApplyError::PersistedValueTooLarge).
  - Q8: content digest per component + replay-verified auto-upgrade; incompatible executions stall until upgraded; no version attribute (deployments; V3 migration).
  - Q9: attempt count and next-eligible time persisted in `t_state`; policy per component (`max_retries`, `retry_exp_backoff`); jitter absent (V1 schema; authored.json).
  - Q10: undocumented; source suggests one catch-up tick then next-from-now (cron_worker.rs, inference).

## Sources
- https://obeli.sk/ — front page claims
- https://obeli.sk/features/ — execution log, determinism, cron, structured concurrency
- https://obeli.sk/docs/latest/ — page index (v0.41.5)
- https://obeli.sk/docs/latest/concepts/workflows/ — determinism, join set extensions, blocking strategy
- https://obeli.sk/docs/latest/concepts/workflows/join-sets/ — submit/await-next/get, close semantics
- https://obeli.sk/docs/latest/concepts/workflows/persistent-sleep/
- https://obeli.sk/docs/latest/concepts/structured-concurrency/
- https://obeli.sk/docs/latest/concepts/activities/ — idempotency contract
- https://obeli.sk/docs/latest/concepts/execution-log/ — event kinds, version tokens
- https://obeli.sk/docs/latest/concepts/execution-states/ — states, lock expiry
- https://obeli.sk/docs/latest/concepts/identifiers/ — id formats
- https://obeli.sk/docs/latest/concepts/deployments/ — auto vs by_component_digest
- https://obeli.sk/docs/latest/concepts/cron/
- https://obeli.sk/docs/latest/concepts/webhook-endpoints/
- https://obeli.sk/docs/latest/concepts/runtime-support/
- https://obeli.sk/docs/latest/concepts/wit-reference/ (general WIT only)
- https://obeli.sk/docs/latest/configuration/ — server/deployment fields and defaults
- https://obeli.sk/docs/latest/cli/
- https://obeli.sk/docs/latest/wasm/getting-started/ — Rust example
- https://obeli.sk/docs/latest/wasm/rust-components/ — build targets, bindings
- https://obeli.sk/docs/latest/patterns/generation-reconciler/ — singleton per key pattern
- https://obeli.sk/docs/latest/migrating-to-0.41/
- https://github.com/obeli-sk/obelisk — README (pre-release warning, persisted value limit)
- https://raw.githubusercontent.com/obeli-sk/obelisk/main/ROADMAP.md
- https://raw.githubusercontent.com/obeli-sk/obelisk/main/crates/db-sqlite/migrations/{V1__initial,V3__workflow_auto_locking,V12__lifecycle,V13__join_set_response_seq,V14__delay_expires_at_index,V18__execution_tombstones,V21__system_events}.sql
- https://raw.githubusercontent.com/obeli-sk/obelisk/main/assets/schemas/toml/authored.json — deployment.toml defaults
- https://raw.githubusercontent.com/obeli-sk/obelisk/main/server-sqlite.toml
- https://raw.githubusercontent.com/obeli-sk/obelisk/main/crates/wasm-workers/src/workflow/event_history.rs — NondeterminismDetected sites
- https://raw.githubusercontent.com/obeli-sk/obelisk/main/crates/wasm-workers/src/cron/cron_worker.rs — tick logic
- https://raw.githubusercontent.com/obeli-sk/obelisk/main/wit/obelisk_workflow@7.0.0/obelisk_workflow@7.0.0.wit, .../wit/obelisk_types@6.0.0/obelisk_types@6.0.0.wit
- https://github.com/obeli-sk/obelisk/releases and https://api.github.com/repos/obeli-sk/obelisk — versions, dates, stars, license
- https://github.com/obeli-sk/obelisk-templates/blob/main/fibo/workflow/README.md
- Failed: https://obelisk.dev/ (DNS not found; the site is obeli.sk)
