# DBOS Transact

Researched: 2026-09-16. Scope: DBOS Transact Python (`dbos-transact-py` main branch, latest release 3.0.0 published 2026-09-16), TypeScript (`dbos-transact-ts` main, latest release v5.0 published 2026-09-16), Go docs and the Go SQLite driver package page; docs.dbos.dev as of today. Schema read from `dbos/_schemas/system_database.py` (Python) and `src/system_database.ts` / `src/sysdb_migrations/internal/migrations.ts` (TS, path only).

## Summary
DBOS Transact is an application library (Python, TypeScript, Go, Java) that makes ordinary functions durable by checkpointing every workflow input and step output into a "system database" (Postgres, or SQLite for prototyping). There is no separate orchestration server: the process that runs your code also runs the recovery scan, the queue pollers and the cron scheduler, all against the database. It optimizes for "add durability to an existing app with one decorator" and for horizontally scaled multi-process deployments sharing one Postgres; an optional hosted control plane (Conductor / DBOS Console) adds cross-executor failover and a UI. The Python SDK is the reference here; TS mirrors it.

## Deployment and process model
- Library only. `DBOS(config=...)` then `DBOS.launch()` in your process; on launch the library runs migrations (unless `run_migrations=False`), starts queue polling threads, the schedule loop, and a startup recovery thread ([`_dbos.py` L623-637](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_dbos.py): "Recover local workflows if not using a recovery service ... Recovering {n} workflows from application version {v}").
- Anything must be running for progress: workflows execute on threads/coroutines inside the launched process; a sleeping workflow keeps its thread. Nothing progresses while no process is launched.
- Identity of a process: `executor_id` (config field or `DBOS__VMID` env; default `"local"`, [`_utils.py` L125](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_utils.py)) and `application_version` (config, `DBOS__APPVERSION`, or an MD5 hash of the sorted source of all registered workflows, [`_dbos.py` compute_app_version](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_dbos.py): "An application's version is computed from a hash of the source of its workflows ... if the app's workflows are updated (which would break recovery), its version changes.").
- Single node: "When restarting an application on a single server without Conductor, DBOS automatically recovers all PENDING workflows from before the restart" (https://docs.dbos.dev/production/workflow-recovery). Distributed: assign per-process executor IDs so only your own PENDING workflows are recovered, or use Conductor which marks a disconnected executor DEAD after a 60 s timeout and "signals another executor to recover its workflows" (same page; https://docs.dbos.dev/production/conductor).
- SQLite: Python defaults to `sqlite:///[application_name].sqlite` when no `system_database_url` is given (https://docs.dbos.dev/python/reference/configuration); "for production, we recommend using Postgres" because a file cannot be shared across servers (https://docs.dbos.dev/python/tutorials/database-connection, via search snippet). Implementation: `_sys_db_sqlite.py` forces `isolation_level="IMMEDIATE"`, `PRAGMA busy_timeout=30000`, `PRAGMA foreign_keys=ON`, runs its own numbered migrations, and refuses attribute (JSONB) filters ("Filtering workflows by attributes is not supported on SQLite"). Go has `dbos/driver/sqlite` "backed by the pure-Go modernc.org/sqlite driver" (https://pkg.go.dev/github.com/dbos-inc/dbos-transact-golang/dbos/driver/sqlite). TS SQLite support: unverified (not checked).

## Execution model
Named-step checkpointing driven by a positional counter, with re-execution from the top. "DBOS restarts each interrupted workflow by calling it with its checkpointed inputs. As the workflow re-executes, it checks before each step if that step's output is checkpointed in Postgres. If there is a checkpoint, the step returns the checkpointed output instead of executing." (https://docs.dbos.dev/architecture). Cost: "one database write per step ... plus two additional database writes per workflow".

Determinism rule, verbatim (https://docs.dbos.dev/python/tutorials/workflow-tutorial): "A workflow function must be **deterministic**: if called multiple times with the same inputs, it should invoke the same steps with the same inputs in the same order." Non-deterministic operations (DB access, API calls, random numbers, timestamps) must be in steps. Guarantees, verbatim: "Workflows always run to completion", "Steps are tried at least once but are never re-executed after they complete", "Transactions commit exactly once". Uncaught exceptions set status `ERROR` with no recovery.

What counts as a step (https://docs.dbos.dev/python/tutorials/step-tutorial): "You should make a function a step if you're using it in a DBOS workflow and it performs a nondeterministic operation." "You cannot call, start, or enqueue workflows from within steps." A step calling a step is folded into the caller's checkpoint. Go: "Each `Go` call is assigned a deterministic step ID, ensuring steps execute in the same order during recovery" (https://docs.dbos.dev/golang/tutorials/workflow-tutorial).

## Step and operation identity
- Identity = `(workflow_uuid, function_id)` where `function_id` is a per-workflow integer counter incremented on every operation: step, child workflow start, `DBOS.sleep`, send/recv, set/get event ([`_context.py` L203-224](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_context.py)). System-tables doc: "Monotonically increasing ID of the step (starts from 0, or from 1 in Python)" (https://docs.dbos.dev/explanations/system-tables). The function name is stored alongside and checked on replay.
- Mismatch: `_check_operation_execution_txn` raises `DBOSUnexpectedStepError`: "During execution of workflow {id} step {n}, function {recorded} was recorded when {expected} was expected. Check that your workflow is deterministic." ([`_error.py`](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_error.py), [`_sys_db.py` L3190-3197](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py)). Same name at the same position with different inputs is NOT detected; inputs are not fingerprinted.
- Unreached recorded steps: nothing checks for them. Because identity is positional, a step the new code no longer reaches shifts every later position; if the shifted name differs you get `DBOSUnexpectedStepError`, if it happens to match you silently get the wrong recorded value. The upgrade doc defines "a breaking change is any change in what steps run or the order in which steps run" (https://docs.dbos.dev/python/tutorials/upgrading-workflows).
- Child workflows: the child's ID is `parent_workflow_id + "-" + function_id` unless set explicitly ([`_context.py` L221-224](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_context.py)), and the parent records `child_workflow_id` in its `operation_outputs` row.
- Workflow ID as idempotency key: "An assigned workflow ID acts as an idempotency key: if a workflow is called multiple times with the same ID, it executes only once" (`with SetWorkflowID("very-unique-id"): example_workflow()`, https://docs.dbos.dev/python/tutorials/workflow-tutorial). Insert conflict is `DBOSWorkflowConflictIDError` / `DBOSConflictingWorkflowError` ("Conflicting workflow invocation with the same ID") when the same ID is reused for a different workflow function.

### Operation lifecycle as implemented (source-derived)
For a step call inside a workflow ([`_core.py` L2236-2300](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_core.py), [`_sys_db.py` call_function_as_step](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py)):
1. `ctx.function_id += 1`; `check_operation_execution(workflow_id, function_id, step_name)` reads `operation_outputs`. Hit with output: return it. Hit with error: re-raise the deserialized error. Name mismatch: `DBOSUnexpectedStepError`.
2. Miss: run the user function on the calling thread, retrying in-process per the decorator's `max_attempts`/`backoff_rate`, each delay capped at 3600 s (`max_retry_interval_seconds`).
3. `record_operation_result` inserts one row `(workflow_uuid, function_id, function_name, output | error, started_at_epoch_ms, serialization)`. Insert conflicts (`DBOSWorkflowConflictIDError`) are tolerated for sleep records.
There is no "attempt admitted" row before execution; the first durable trace of a step is its outcome. Inference from the schema, not tested: two processes executing the same PENDING workflow would both run the step, and the second `operation_outputs` insert would hit the `(workflow_uuid, function_id)` primary key.

For a child workflow start: `record_child_workflow(parent_id, function_id, child_id)` writes the child's ID into the parent's `operation_outputs` row (`child_workflow_id`) so replay returns a handle instead of starting a second child ([`_core.py` L1414-1474](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_core.py)). Workflow status rows are inserted before execution (`_insert_workflow_status`), with `recovery_attempts = 1` for direct starts and 0 for enqueued ones ("only the queue's claim counts a dispatch").

For a queued workflow: `start_queued_workflows` runs one transaction per poll that (a) computes remaining rate-limit slots from persisted `started_at_epoch_ms`, (b) counts local and global running workflows, (c) claims `ENQUEUED` rows ordered by priority/creation, setting `status = PENDING, executor_id = <me>, application_version = <me>, started_at_epoch_ms = now, recovery_attempts + 1` ([`_sys_db.py` L4425-4860](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py)). Postgres uses REPEATABLE READ or SERIALIZABLE when a budget is shared; SQLite relies on IMMEDIATE transactions serializing writers.

Enqueue options persisted per workflow: `deduplication_id` (a second enqueue with the same ID on the same queue raises `DBOSQueueDeduplicatedError` while the first is active), `priority` (0 highest), `queue_partition_key`, and `delay_until_epoch_ms` (status `DELAYED`).

## Persistence schema
Source: [`dbos/_schemas/system_database.py`](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_schemas/system_database.py) (schema name `dbos`, TS: `src/sysdb_migrations/internal/migrations.ts`). Tables:
- `workflow_status(workflow_uuid PK, status, name, authenticated_user, assumed_role, authenticated_roles, output, error, executor_id, created_at, updated_at, application_version, application_id, class_name, config_name, recovery_attempts, queue_name, workflow_timeout_ms, workflow_deadline_epoch_ms, started_at_epoch_ms, deduplication_id, inputs, priority, queue_partition_key, forked_from, was_forked_from, owner_xid, parent_workflow_id, serialization, delay_until_epoch_ms, rate_limited, completed_at, attributes JSONB, schedule_name, debounce_deadline_epoch_ms, is_debounced, application_name)`. Status enum: `PENDING, SUCCESS, ERROR, MAX_RECOVERY_ATTEMPTS_EXCEEDED, CANCELLED, ENQUEUED, DELAYED` ([`_sys_db.py` L117-127](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py)).
- `workflow_input(workflow_uuid, inputs, retention_timestamp)`, `workflow_output(workflow_uuid, output, error, retention_timestamp)` (payloads split out for retention sweeps).
- `operation_outputs(workflow_uuid, function_id, function_name, output TEXT, error TEXT, child_workflow_id, started_at_epoch_ms, completed_at_epoch_ms, serialization, application_name, retention_timestamp; PK(workflow_uuid, function_id))`. Step outputs are serialized inline as TEXT; no size limit or reference mechanism found.
- `notifications`, `workflow_events` (+ immutable `workflow_events_history`), `streams` for send/recv, set/get event, streams.
- `workflow_schedules(schedule_id, schedule_name UNIQUE, workflow_name, workflow_class_name, schedule, status ACTIVE/PAUSED, context, last_fired_at, automatic_backfill, cron_timezone, queue_name, application_name)`.
- `queues(queue_id, name UNIQUE, concurrency, worker_concurrency, rate_limit_max, rate_limit_period_sec, priority_enabled, partition_queue, partition_concurrency, partition_worker_concurrency, partition_rate_limit_max, partition_rate_limit_period_sec, polling_interval_sec, ...)` (queue config lives in the DB; "you can change a queue's configuration at runtime without redeploying", https://docs.dbos.dev/python/tutorials/queue-tutorial).
- `application_versions(version_id, version_name, version_timestamp, ...)` for "latest version" routing.
Retention: rows-threshold garbage collection (default 1M rows per the DBOS Cloud retention doc, search snippet only, page not fetched) and `garbage_collect` in `_sys_db.py`.

TypeScript specifics checked in source only: the same `workflow_status`/`operation_outputs` model, `getPendingWorkflows` filtered by `status, executor_id, application_version` with the comment "executor_id defaults to \"local\", so it collides across applications" ([`system_database.ts` L1579-1612](https://raw.githubusercontent.com/dbos-inc/dbos-transact-ts/main/src/system_database.ts)); migrations live in `src/sysdb_migrations/internal/migrations.ts` (48 KB, not read). TS v5.0 release notes (search snippet) describe retention deleting from `workflow_input`, `workflow_output` and `operation_outputs` in batches ordered by creation timestamp.

## Crash recovery of in-flight work
- Detection: on launch, `get_pending_workflows(executor_id, app_version)` selects `status = PENDING AND executor_id = ? AND application_version = ?` ([`_sys_db.py` L2418-2439](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py)). Each is re-enqueued (`PENDING -> ENQUEUED` on `_dbos_internal_queue`) rather than executed directly, "so that every recovered workflow starts through the queue's atomic ENQUEUED->PENDING dequeue. That handoff admits exactly one runner, which makes duplicate recovery requests idempotent." ([`_recovery.py`](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_recovery.py)). `reenqueue_for_recovery` only matches rows whose `executor_id` is in the declared-dead set, so a live executor that already claimed the row is not disturbed.
- A step that was running when the process died has no `operation_outputs` row, so it is executed again: "if a workflow fails while executing a step, it retries the step during recovery. However, once a step completes and is checkpointed, it is never re-executed" (https://docs.dbos.dev/architecture). The user cannot declare a step "uncertain / manual"; the only lever is making the step idempotent.
- Recovery budget: each dequeue increments `recovery_attempts`; when `recovery_attempts > max_recovery_attempts + 1` (default `DEFAULT_MAX_RECOVERY_ATTEMPTS = 100`, [`_registrations.py` L10](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_registrations.py)) the workflow is dead-lettered as `MAX_RECOVERY_ATTEMPTS_EXCEEDED` ([`_core.py` L1220-1232](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_core.py)).
- No leases or heartbeats in the library itself; liveness is either "same executor restarted" or Conductor's 60 s disconnect timeout. Known race in the Java port: "Launch-time recovery can re-execute a workflow that is already running in-process (duplicate execution)" (https://github.com/dbos-inc/dbos-transact-java/issues/491; fix PR #493 "Stop recovery adopting workflows this executor is already running"). The Python re-enqueue design above is the same class of fix.
- Cancellation preempts "at the beginning of its next step"; a step interrupted by cancellation records no outcome ("let the step be re-run on resume", [`_core.py` L2280-2283](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_core.py)).

## Retries, backoff, timeouts
- Step retries are declared on the decorator: `@DBOS.step(retries_allowed=True, interval_seconds=1.0, max_attempts=3, backoff_rate=2.0, should_retry=...)`; delay is `min(interval_seconds * backoff_rate**attempt, 3600)` computed in-process ([`_core.py` L2242-2262](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_core.py)). Exhaustion raises `DBOSMaxStepRetriesExceeded` to the workflow.
- Not persisted: the `operation_outputs` row is written only when the step finally succeeds or finally fails (`record_step_result`). Attempt count and next-retry time live in the thread; a crash mid-backoff resets the allowance and the delay on recovery (inference from source; no doc statement). No jitter on step retries.
- Workflow-level: `SetWorkflowTimeout(seconds)` persisted as `workflow_deadline_epoch_ms`; "When the timeout expires, the workflow and all its children are cancelled" (workflow tutorial). Queue-level retry of a whole workflow only via `recovery_attempts` above.
- Classification: `should_retry: Callable[[BaseException], bool]` on the step is the only retryable/terminal hook; no error-class taxonomy.

## Flow control
- Queues are the unit: `DBOS.register_queue("name", worker_concurrency=5, global_concurrency=10, limiter={"limit": 50, "period": 30}, partition_concurrency=1, ...)`; `SetEnqueueOptions(priority=..., deduplication_id=..., queue_partition_key=...)` (https://docs.dbos.dev/python/tutorials/queue-tutorial).
- Persistence: queue config in `queues`; per-workflow `priority`, `deduplication_id`, `queue_partition_key`, `rate_limited`, `started_at_epoch_ms` in `workflow_status`. "Rate limits are global across all DBOS processes using this queue" — implemented by counting `workflow_status` rows with `rate_limited = TRUE AND started_at_epoch_ms > now - period` under REPEATABLE READ/SERIALIZABLE on Postgres ([`_sys_db.py` L4425-4481](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py)); so the rolling window is derived from persisted start times, not from an in-memory token bucket. `worker_concurrency` is a local running count; `global_concurrency` counts `PENDING` rows on the queue, hence the warning: "any `PENDING` workflow on the queue counts toward the limit, including workflows from previous application versions."
- Keyed by queue name and optional partition key; there is no per-host or per-URL key other than making a queue/partition per host. Debounce exists (`debounce_deadline_epoch_ms`, `_debouncer.py`; not researched further).

## Fan-out, child workflows, map
- Primitives: call a workflow from a workflow (child; ID `parent-<function_id>`), `DBOS.start_workflow` (handle), enqueue to a queue and collect handles, `handle.get_result()`. No `map` primitive; the work set is whatever the deterministic workflow enumerates, recorded implicitly as consecutive `operation_outputs` rows (one per child start) and the parent link `parent_workflow_id`.
- Ordering and failure policy are the user's loop; a child failure surfaces as an exception on `get_result()`.
- Fork: "DBOS generates a new workflow with a new workflow ID, copies to that workflow the original workflow's inputs and all its steps up to the selected step, then begins executing the new workflow from the selected step" (https://docs.dbos.dev/python/tutorials/workflow-management); `forked_from`/`was_forked_from` columns.

## Versioning and code change policy
Two strategies (https://docs.dbos.dev/python/tutorials/upgrading-workflows):
- Versioning: "All workflows are tagged with the application version on which they started." "When DBOS tries to recover workflows, it only recovers workflows whose version matches the current application version." Default version = MD5 of workflow source; override via `application_version`. Old workflows need an old-version process (blue-green); `DBOS.set_latest_application_version()` / `get_latest_application_version()` steer enqueued and scheduled work to a version.
- Patching (`enable_patching: True`): `if DBOS.patch("use-baz"): baz() else: foo()`; "`DBOS.patch()` attempts to insert a 'patch marker' at its current point in your workflow history" — returns True for new workflows, False for old; later `DBOS.deprecate_patch()`.
- Failure mode: if neither is done, "workflows will throw a `DBOSUnexpectedStepError`". So the gate is a build digest by default (source hash), fail-stop on detected mismatch, no warn-and-continue.

## Schedules, timers, missed runs
- Durable sleep: `DBOS.sleep(seconds)` writes an `operation_outputs` row named `DBOS.sleep` whose output is the absolute wake time before sleeping; on replay it sleeps only the remainder ([`_sys_db.py` record_sleep](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py)). Docs: "DBOS saves the wakeup time in the database so that even if the workflow is interrupted and restarted multiple times while sleeping, it still wakes up on schedule."
- Schedules: `DBOS.create_schedule(schedule_name=, workflow_fn=, schedule="*/5 * * * *", context=...)`; the workflow receives `(scheduled_time: datetime, context)`. Rows in `workflow_schedules`; a thread per active schedule computes the next fire with croniter, adds jitter of up to 10 % of the wait capped at 10 s, and enqueues a workflow with ID `f"sched-{schedule_name}-{next_exec_time.isoformat()}"` after checking it does not already exist ([`_scheduler.py` L57-95](https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_scheduler.py)); the scheduled instant is therefore the dedup identity across processes.
- Missed runs: by default nothing is done for fires that happened while no process was up. `DBOS.backfill_schedule(name, start, end)` enqueues every cron instant in the window ("Already-executed times are automatically skipped"), and `automatic_backfill=True` makes the scheduler backfill from `last_fired_at` to now "whenever your application starts or a paused schedule is resumed" (https://docs.dbos.dev/python/tutorials/scheduled-workflows; `_scheduler.py` L304-315). Backfill is a full catch-up burst; there is no "one current run" mode other than leaving backfill off. Overlap: no overlap policy; concurrency comes from putting the schedule on a queue (`queue_name`).

## Cancellation and late results
- `DBOS.cancel_workflow(id)`: status `CANCELLED`; "If the workflow is currently executing, cancelling it preempts its execution (interrupting it at the beginning of its next step). If the workflow is enqueued, cancelling removes it from the queue." A running step is not interrupted; it finishes, then the workflow stops. `resume_workflow` continues from the last completed step; `fork_workflow` starts a copy from a chosen step.
- No documented compensation hooks; timeouts cancel the workflow tree.

## Observability and tooling
- CLI (https://docs.dbos.dev/python/reference/cli): `dbos workflow list|get|steps|cancel|resume|fork`, `dbos workflow queue list`, `dbos migrate`, `dbos reset` ("deleting metadata about past workflows and steps"), `dbos start`, `dbos init`, `dbos rename-application`; all accept `-s/--sys-db-url` and `--schema`.
- Programmatic: `DBOS.list_workflows`, `list_workflow_steps`, `cancel_workflow`, `resume_workflow`, `fork_workflow`, `recover_pending_workflows(executor_ids)`; an admin HTTP server (`run_admin_server`) exposes recovery endpoints (not fetched).
- Conductor + DBOS Console (hosted; "DBOS Teams" plan mentioned for a metadata-only mode): trace timeline of workflow, steps and children; cancel/resume/fork by clicking a step; retention policy page; executor status (https://docs.dbos.dev/production/conductor). Cross-run cache explanation: none (no cache concept).

## API ergonomics
Minimal example, verbatim from https://docs.dbos.dev/python/tutorials/workflow-tutorial:
```python
@DBOS.step()
def step_one():
    print("Step one completed!")

@DBOS.step()
def step_two():
    print("Step two completed!")

@DBOS.workflow()
def workflow():
    step_one()
    step_two()
```
Go equivalent (https://docs.dbos.dev/golang/tutorials/workflow-tutorial): `dbos.RunAsStep(ctx, stepOne)` inside `func workflow(ctx dbos.Context, _ string) (string, error)`.

Delightful:
- No step IDs to invent; a step is a decorated function called normally. Steps stay directly callable outside workflows.
- Queues, priorities, dedup keys, partition keys and rate limits are one `register_queue` call and are visible in SQL.
- `SetWorkflowID` gives idempotent, singleton-per-key starts with a one-line context manager.
- Durable sleep, timeouts, fork/resume and the CLI come for free from the same table.

Painful / footguns:
- Positional identity: any insertion, removal or reordering of a step in code is a "breaking change" that needs a patch marker or a version bump; a same-name/different-input step is silently accepted.
- Retries of a step are not persisted (source inference above); a 429 cooldown chosen by the step is lost on crash.
- Global concurrency counts stale `PENDING` rows of old versions (documented warning).
- Executor ID defaults to `"local"`, so two different apps on one system DB collide unless `application_name` is set (source comment: "executor_id defaults to 'local', so it collides across applications"); TS issue #1356 reports a shutdown path that rewrites `executor_id` to `local` and prevents recovery (title from search; not fetched).
- Coroutine workflows must use the async context methods; sync steps run on a thread pool.

Scraping pipeline sketch: one `refresh_source(source)` workflow per source started `with SetWorkflowID(f"refresh/{source.id}/{date}")`; `fetch_page` as a step with `retries_allowed=True` enqueued onto a per-host queue with `limiter={"limit": n, "period": 60}` and `worker_concurrency`; parse/enrich as steps; publish as the last step. The user must write: the per-host queue registration, the loop over pages (deterministic, page list must be recorded by a step first), collection of handles, snapshot storage (outputs are inline TEXT), and any "do not repeat this fetch" policy beyond idempotency.

## Known limitations
- Retry/backoff state for steps is in-memory (source; no doc statement).
- Unreached or reordered steps are not detected as such; only a name mismatch at a position is (`DBOSUnexpectedStepError`).
- Missed schedule fires require explicit or automatic backfill; there is no "run once for the whole gap" option (scheduled-workflows doc + `_scheduler.py`).
- SQLite backend: single file, no attribute filters, Postgres recommended for production (configuration reference; `_sys_db_sqlite.py`).
- Duplicate execution race at launch in the Java port (#491); Python avoids it by re-enqueueing through the queue claim.
- Conductor / Console are a hosted service, not part of the OSS library.
- Go GC lags Python ("Go garbage collection is behind Python: admin endpoint is a stub, delete is unbatched", https://github.com/dbos-inc/dbos-transact-golang/issues/469, title from search; not fetched).

## Relevance to Flowyard
- Borrow: the schema shape (`workflow_status` + `operation_outputs` keyed by run and operation, payload tables split for retention); recording the sleep deadline as an operation before sleeping; `sched-{name}-{instant}` as the dedup identity of a scheduled run; "recovery re-enqueues through the same claim path as normal starts" to make duplicate recovery idempotent; workflow ID as idempotency key; the fork-from-step operation; the CLI verbs list/get/steps/cancel/resume/fork; rate limits derived from persisted start timestamps rather than an in-memory bucket.
- Avoid: positional (`function_id`) identity as the primary key of an operation; source-hash-as-version without an explicit override being the norm; unpersisted step retry state; backfill-only missed-run policy; global concurrency counted from stale rows.
- Q1 (unreached steps): not detected; positional shift causes `DBOSUnexpectedStepError` or silent misattribution (source + upgrading doc).
- Q2 (singleton per key): yes, workflow ID is the key; a second call with the same ID "executes only once" (workflow tutorial); queue `deduplication_id` is the enqueue-time variant.
- Q3 (daemon vs one-shot): long-running process assumed; recovery, queues and schedules are background threads started by `DBOS.launch()`.
- Q4 (inline vs queue): steps run inline in the workflow's thread; only workflow starts go through the persisted queue (`ENQUEUED -> PENDING`).
- Q5 (rate limits): persisted via `queues` config and `workflow_status.started_at_epoch_ms`; keyed by queue name and partition key, not host.
- Q6 (granularity): "make a function a step if ... it performs a nondeterministic operation"; pure computation is left outside steps.
- Q7 (large outputs): inline TEXT in `operation_outputs`; no limit or reference type documented.
- Q8 (code-change gate): version tag = source hash by default, explicit `application_version` optional, patch markers for in-place changes; mismatch fails with `DBOSUnexpectedStepError`.
- Q9 (retry state): workflow `recovery_attempts` persisted; step attempts and next-retry time not persisted; no jitter on step retries; scheduler jitter only.
- Q10 (missed runs): nothing by default; `automatic_backfill=True` is a catch-up burst of every missed instant.

## Sources
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_schemas/system_database.py (schema)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db.py (status enum, pending query, step check, rate limit, sleep, recovery re-enqueue)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_recovery.py (startup recovery)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_sys_db_sqlite.py (SQLite backend)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_core.py (step retry loop, max recovery check)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_context.py (function_id counter, child ID)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_error.py (error texts)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_scheduler.py (schedule loop, backfill)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_dbos.py (launch recovery, app version hash)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-py/main/dbos/_utils.py, _registrations.py, _dbos_config.py (defaults)
- https://raw.githubusercontent.com/dbos-inc/dbos-transact-ts/main/src/system_database.ts (TS executor/version filters; migrations path)
- https://docs.dbos.dev/python/tutorials/workflow-tutorial (determinism, guarantees, IDs, sleep, timeouts, cancellation)
- https://docs.dbos.dev/python/tutorials/step-tutorial (retry params, what is a step)
- https://docs.dbos.dev/python/tutorials/queue-tutorial (queues, limiter, warning)
- https://docs.dbos.dev/python/tutorials/scheduled-workflows (schedules, backfill)
- https://docs.dbos.dev/python/tutorials/workflow-management (list/cancel/resume/fork)
- https://docs.dbos.dev/python/tutorials/upgrading-workflows (patching vs versioning)
- https://docs.dbos.dev/python/reference/configuration (SQLite default URL, fields)
- https://docs.dbos.dev/python/reference/cli (CLI)
- https://docs.dbos.dev/architecture (recovery model)
- https://docs.dbos.dev/production/workflow-recovery (single server, executor IDs, Conductor timeout)
- https://docs.dbos.dev/production/conductor (Conductor/Console)
- https://docs.dbos.dev/explanations/system-tables (table doc)
- https://docs.dbos.dev/golang/tutorials/workflow-tutorial (Go example)
- https://pkg.go.dev/github.com/dbos-inc/dbos-transact-golang/dbos/driver/sqlite (Go SQLite driver)
- https://github.com/dbos-inc/dbos-transact-java/issues/491 (duplicate execution at launch)
- Search snippets only (page not fetched): https://docs.dbos.dev/python/tutorials/database-connection, https://docs.dbos.dev/production/dbos-cloud/retention, https://github.com/dbos-inc/dbos-transact-ts/issues/1356, https://github.com/dbos-inc/dbos-transact-golang/issues/469
- Tried and failed (404): https://docs.dbos.dev/production/self-hosting/workflow-recovery, https://docs.dbos.dev/production/self-hosting/conductor, https://docs.dbos.dev/production/self-hosting/workflow-management
