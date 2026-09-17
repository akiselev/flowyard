# Dagster

Researched: 2026-09-16. Scope: Python; docs.dagster.io (markdown read from `dagster-io/dagster` `master` `docs/docs/**`, plus `examples/docs_snippets`), and `master` source under `python_modules/dagster/dagster/_core/storage/{runs,event_log,schedules}/schema.py`, `_core/storage/dagster_run.py`, `_core/events/__init__.py`, `_core/scheduler/scheduler.py`, `_scheduler/scheduler.py`. Dagster+ (cloud) features noted only where the docs distinguish them.

## Summary
Dagster is an asset-oriented orchestrator: "code is the source of truth on what data assets should exist and how those assets are computed". Users declare assets (functions with upstream `deps`), optionally partitioned, plus checks, schedules and sensors; Dagster compiles job runs into explicit step plans and records everything in an append-only event log. It is run by data platform teams with a webserver, a daemon and one or more code-location servers. It optimizes for lineage, staleness/health of persisted data, and operational control of many runs (queues, pools, backfills), not for durable in-process control flow.

## Deployment and process model
- Long-running services: `dagster-webserver` (UI/GraphQL), `dagster-daemon` ("Several Dagster features, like schedules, sensors, and run queueing, require a long-running `dagster-daemon` process"), and code-location gRPC servers loading your definitions. `dagster dev` runs webserver + daemon locally; `dagster-daemon run` alone; `dg dev` in the new `dg` CLI. Daemons: Scheduler, Run queue (only with `QueuedRunCoordinator`), Sensor, Run monitoring. "Each daemon periodically writes a heartbeat to your instance storage."
- Instance config is `$DAGSTER_HOME/dagster.yaml`. Default storage is SQLite: `history/runs.db` ("SQLite database file that contains information about runs"), `history/[run_id].db` ("per-run event logs"), `schedules/` DB, `storage/[run_id]/compute_logs` for stdout/stderr (oss-instance-configuration). Postgres/MySQL via `storage.postgres`/`storage.mysql`.
- Runs: submitted to the run coordinator (`DefaultRunCoordinator` launches immediately in-process; `QueuedRunCoordinator` puts it in a DB-backed queue the daemon drains), then a run launcher (`DefaultRunLauncher`: "Spawns a new process on the same node as the job's code location"; Docker; K8s) starts a *run worker*; the executor runs steps ("By default, Dagster will run the job using the `multiprocess_executor` - that means each step in the job runs in its own process").
- Single-node story: `dagster dev` with SQLite; everything else (queue, monitoring, catch-up) needs the daemon alive.

## Execution model
- Explicit persisted structure. A job (op graph or asset selection) is snapshotted at run creation (`snapshots` table: `snapshot_body` LargeBinary, `snapshot_type`), turned into an execution plan of step keys, and executed by an executor that emits typed events into the event log: `STEP_START`, `STEP_OUTPUT`, `STEP_INPUT`, `STEP_SUCCESS`, `STEP_FAILURE`, `STEP_SKIPPED`, `STEP_UP_FOR_RETRY`, `STEP_RESTARTED`, `ASSET_MATERIALIZATION(_PLANNED)`, `ASSET_OBSERVATION`, `ASSET_CHECK_EVALUATION(_PLANNED)`, `RUN_ENQUEUED/STARTING/START/SUCCESS/FAILURE/CANCELING/CANCELED` (`_core/events/__init__.py`).
- No replay of user code and no checkpoints inside an op: a step either succeeds (its outputs handed to an IO manager) or fails. Resume = a *new run* that references the old one: "By default, retries will re-execute from failure (tag value `FROM_FAILURE`). This means that any successful ops will be skipped, but their output will be used for downstream ops." Caveat: "`FROM_FAILURE` requires an I/O manager that can access outputs from other runs" (run-retries). `FROM_ASSET_FAILURE` uses "persisted asset materialization records from the event log and automatically exclude already-materialized assets during retry".
- Determinism rules: none on op bodies. Dynamic graphs are the only runtime-shaped structure (below).
- Run statuses (`DagsterRunStatus`): `QUEUED`, `NOT_STARTED`, `MANAGED`, `STARTING`, `STARTED`, `SUCCESS`, `FAILURE`, `CANCELING`, `CANCELED`; finished = SUCCESS/FAILURE/CANCELED.

## Step and operation identity
- Step key = op name in the graph (nested graphs prefix the path); asset key = list of strings (`key_prefix` + name); partition key string; check name per asset key. Dynamic fan-out clones are "identified using the associated `mapping_key`".
- Identity is structural and known before execution (the plan is persisted), so "mismatch" is a definitions change: reloading a code location changes the job snapshot; existing runs keep their own snapshot. Partitioned assets additionally get `partition`/`partition_set` columns on the run row and per-partition materialization events.
- Unreached steps: not applicable in the replay sense. A run that is re-executed with an op selection (`*some_op`, `some_op+`) just does not include the other steps; skipped steps emit `STEP_SKIPPED`. Assets materialized by older code stay in the catalog and get an "Unsynced" label when `code_version` or upstream data versions change (asset-versioning-and-caching).

## Persistence schema
From `_core/storage/runs/schema.py`: `runs(id, run_id UNIQUE, snapshot_id FK, pipeline_name, mode, status, run_body TEXT, partition, partition_set, create_timestamp, update_timestamp, start_time FLOAT, end_time FLOAT, backfill_id)`, `run_tags(run_id FK CASCADE, key, value)`, `snapshots(snapshot_id, snapshot_body BLOB, snapshot_type)`, `daemon_heartbeats(daemon_type UNIQUE, daemon_id, timestamp, body)`, `bulk_actions(key, status, timestamp, body, action_type, selector_id, job_name)` (backfills), `backfill_tags`, `instance_info`, `kvs(key, value)`, `secondary_indexes`.
From `event_log/schema.py`: `event_logs(id, run_id, event LONGTEXT NOT NULL, dagster_event_type, timestamp, step_key, asset_key, partition)`; `asset_keys(asset_key UNIQUE, last_materialization, last_run_id, asset_details, wipe_timestamp, last_materialization_timestamp, tags, cached_status_data)`; `asset_event_tags`; `dynamic_partitions(partitions_def_name, partition)`; `concurrency_limits(concurrency_key UNIQUE, limit)`; `concurrency_slots(concurrency_key, run_id, step_key, deleted)`; `pending_steps(concurrency_key, run_id, step_key, priority, assigned_timestamp)`; `asset_check_executions(asset_key, check_name, partition, run_id, execution_status "Planned, Success, or Failure", evaluation_event, evaluation_event_timestamp)`.
From `schedules/schema.py`: `jobs`/`instigators(selector_id UNIQUE, status, instigator_type, instigator_body)`, `job_ticks(job_origin_id, selector_id, status, type, timestamp, tick_body)`, `asset_daemon_asset_evaluations`.
- Step outputs are not in the DB. They go through IO managers; the default filesystem IO manager pickles under `local_artifact_storage` (`storage/` "A directory of subdirectories, one for each run"). The event log stores metadata (`STEP_OUTPUT`, materialization metadata, data versions), so large outputs are always referenced. SQLite event logs are one file per run (`SqliteEventLogStorage.path_for_shard: f"{run_id}.db"`) plus an index shard. Migrations via `dagster instance migrate`.

## Crash recovery of in-flight work
- Without monitoring: "If the run worker was able to catch the interrupt, it will mark the run as failed; If the run worker goes down without a grace period, the run could be left hanging in STARTED status."
- Run monitoring daemon (`run_monitoring.enabled`): `start_timeout_seconds` (STARTING too long -> failed), `cancel_timeout_seconds`, `max_runtime_seconds` / `dagster/max_runtime` tag (STARTED too long -> failed), `free_slots_after_run_end_seconds`. Verbatim limits: "Detecting run worker crashes only works when using a run launcher other than the `DefaultRunLauncher`." and resuming a crashed worker "is currently only supported when using: `K8sRunLauncher` with the `k8s_job_executor`; `DockerRunLauncher` with the `docker_executor`" (`max_resume_run_attempts`).
- Whole-run retries also cover crashes: "Run retries also handle the case where the run process crashes or is unexpectedly terminated." (new run, FROM_FAILURE by default). No per-op declaration of idempotent/uncertain; an interrupted step is simply not `STEP_SUCCESS` and reruns.
- Stale slot warning (verbatim): "By default, Dagster does not automatically free concurrency pool slots when a run is cancelled or fails. ... With a pool limit of 1, a single cancelled run will permanently deadlock all future runs for that pool."

## Retries, backoff, timeouts
- Op retries (in the same run): `@dg.op(retry_policy=dg.RetryPolicy(max_retries=3, delay=0.2, backoff=dg.Backoff.EXPONENTIAL, jitter=dg.Jitter.PLUS_MINUS))`; manual `raise dg.RetryRequested(max_retries=1, seconds_to_wait=1) from e`; `dg.Failure(..., allow_retries=False)` to bypass. Policy can be set per op, per invocation (`with_retry_policy`), per job (`op_retry_policy`), or on `define_asset_job(op_retry_policy=...)` ("The maximum retry limit in this case applies to each individual asset in the job, not to the entire run.").
- Persisted: `STEP_UP_FOR_RETRY` and `STEP_RESTARTED` events; the wait itself is in the executor process (not persisted; unverified beyond the event list).
- Run retries (whole run, new run id): `run_retries: {enabled: true, max_retries: 3, retry_on_asset_or_op_failure: false}`; tags `dagster/max_retries`, `dagster/retry_strategy` (`FROM_FAILURE` default, `ALL_STEPS`, `FROM_ASSET_FAILURE`). Footgun verbatim: "the op retry count will reset for each retried run."
- Timeouts: no default ("By default, Dagster jobs have no automatic timeout"); `dagster/max_runtime` tag + run monitoring.

## Flow control
- Run queue (`QueuedRunCoordinator`): `max_concurrent_runs`, `tag_concurrency_limits` (key or key/value), `dagster/priority` tag (integer string, negatives allowed), FIFO otherwise; "a run blocked by tag concurrency limits won't block runs submitted after it".
- Concurrency pools (cross-run, DB-backed `concurrency_limits`/`concurrency_slots`/`pending_steps`): `@dg.asset(pool="database")`, limit via UI or `dagster instance concurrency set database 1`; `concurrency.pools.default_limit`; `granularity: 'run'` to limit runs containing pooled ops instead of ops; dequeue order not strict FIFO under pools.
- In-run executor limits: `max_concurrent` and per-run `tag_concurrency_limits` ("These limits are only applied on a per-run basis").
- No rate limiter (only concurrency), no debounce; batching is modeled with partitions/backfill policies.

## Fan-out, child workflows, map
- Dynamic outputs (ops only): `DynamicOut` + `yield DynamicOutput(value, mapping_key=...)`, then `.map(fn)` and `.collect()`; "The ops downstream of a dynamic output will be cloned for each dynamic output, and identified using the associated `mapping_key`." Usable in assets via graph-backed assets ("Failed sub-pipelines can be retried independently").
- Partitions: `DailyPartitionsDefinition`, `StaticPartitionsDefinition`, `MultiPartitionsDefinition`, `DynamicPartitionsDefinition` (sensor adds partitions at runtime). Backfills: "if you launch a backfill that covers `N` partitions, Dagster will launch `N` separate runs" or `BackfillPolicy.single_run`; backfills are `bulk_actions` rows and need the daemon. "We recommend limiting the number of partitions for each asset to 100,000 or fewer."
- Sensors dedupe with `run_key` ("Dagster will skip processing requests with previously used run keys") and keep a string `cursor`.

## Versioning and code change policy
- Code locations reload on demand ("Reload definitions"); the daemon re-reads `workspace.yaml` without restart. Runs carry a job snapshot, so in-flight runs are unaffected by a reload.
- Assets: `code_version="1"` string; data version = "hashing a code version together with the data versions of any input assets"; user-supplied `DataVersion` via `Output`; observable source assets compute versions for external inputs. Without a code version "Dagster assumes a different code version on every run, which it represents with the run ID." Change => "Unsynced" label; not transitive; "Materialize unsynced" action. Warn-style, never fail.
- No migration tooling for in-flight runs; re-execution of an old run after a definitions change is not documented in the pages read (unverified).

## Schedules, timers, missed runs
- `dg.ScheduleDefinition(name=..., cron_schedule="0 0 * * *", target=[...])`, `execution_timezone`, `build_schedule_from_partitioned_job`. Ticks are rows in `job_ticks`; schedule state in `instigators`. Retention: `retention.schedule.purge_after_days`.
- Missed ticks: `DagsterDaemonScheduler(max_catchup_runs=5)` (`DEFAULT_MAX_CATCHUP_RUNS = 5`): "For partitioned schedules, controls the maximum number of past partitions for each schedule that will be considered when looking for missing runs ... This parameter will not be checked for schedules without partition sets ... only the most recent execution time will be considered for those schedules." and "the scheduler will never launch a run from a time before the schedule was turned on".
- Overlap: no built-in skip-if-running; docs suggest `dagster/max_runtime` for "Scheduled jobs that should not overlap with subsequent ticks", pools with run granularity, or run-queue tag limits.
- Declarative automation reactivation flood: "the default `AutomationCondition.eager` condition triggers execution for every partition that became eligible while the sensor was off ... this can produce thousands of unwanted runs"; fix is a time-aware condition.
- Timers: no durable sleep; sensors poll (`minimum_interval_seconds`).

## Cancellation and late results
- `CANCELING` -> termination signal to the run worker -> `CANCELED`; `cancel_timeout_seconds` forces it. Interrupted steps raise `DagsterExecutionInterruptedError`. Slots held by cancelled runs are not released without `free_slots_after_run_end_seconds`. Late step results after cancel: not documented (unverified).

## Observability and tooling
- UI: runs/timeline, asset catalog & lineage, materializations/events per asset, partitions grid, automation (schedules/sensors, tick history with logs), Daemons tab (heartbeats), Concurrency settings, Launchpad (config, tags, op selection), re-execute from failure.
- CLI (`dagster`): `asset list|materialize|wipe|wipe-partitions-status-cache`; `job execute|launch|backfill|list|print|scaffold_config`; `run list|delete|wipe|migrate-repository`; `instance info|migrate|reindex|concurrency`; `schedule list|start|stop|restart|preview|logs|debug|wipe`; `sensor list|start|stop|preview|cursor`; `debug export|import` (run artifacts to a file); `definitions validate`; `dev`; `dagster-daemon run|wipe|debug heartbeat-dump`; `dagster-graphql`. New CLI: `dg dev`, `dg launch --jobs my_job`. Python: `job.execute_in_process()`.
- Cache explanation: asset "Unsynced" reasons (code version changed, deps changed, upstream data version) in the UI sidebar; no CLI equivalent found.

## API ergonomics
Minimal complete example, verbatim from docs snippets used by "Defining assets" and "Schedules" (https://docs.dagster.io/guides/build/assets/defining-assets, https://docs.dagster.io/guides/automate/schedules; files `examples/docs_snippets/docs_snippets/guides/build/assets/data-assets/data-assets/asset_decorator.py` and `.../guides/automate/simple-schedule-example.py`):
```python
import dagster as dg


@dg.asset
def daily_sales() -> None: ...


@dg.asset(deps=[daily_sales], group_name="sales")
def weekly_sales() -> None: ...


@dg.asset(
    deps=[weekly_sales],
    owners=["bighead@hooli.com", "team:roof", "team:corpdev"],
)
def weekly_sales_report(context: dg.AssetExecutionContext):
    context.log.info("Loading data for my_dataset")
```
```python
import dagster as dg


@dg.asset
def customer_data(): ...


@dg.asset
def sales_report(): ...


daily_schedule = dg.ScheduleDefinition(
    name="daily_refresh",
    cron_schedule="0 0 * * *",  # Runs at midnight daily
    target=[customer_data, sales_report],
)

defs = dg.Definitions(schedules=[daily_schedule])
```
Delightful:
- Assets with `deps=[...]` give lineage, staleness and a catalog for free; partitions are one argument (`partitions_def=daily_partitions`, `context.partition_key`).
- `@dg.asset_check(asset=orders)` returning `AssetCheckResult(passed=...)`, `blocking=True` to stop downstream.
- `RetryPolicy(max_retries, delay, backoff, jitter)` is declarative; `RetryRequested` for programmatic cases; `Failure(allow_retries=False)`.
- `pool="database"` on an asset + one CLI command gives a cross-run limit; `run_key` dedup and `cursor` on sensors.
- Backfills as a first-class object with per-partition or single-run policies.

Painful / footguns (with cites):
- Three processes plus `dagster.yaml` before schedules run at all (dagster-daemon; dagster-yaml).
- Op retries and run retries compound ("the op retry count will reset for each retried run") (run-retries).
- `FROM_FAILURE` needs an IO manager readable across runs; the default filesystem manager breaks on separate hosts/containers (run-retries note).
- Crash detection/resume only on non-default launchers; local `DefaultRunLauncher` runs can hang in STARTED (run-monitoring).
- Cancelled runs keep pool slots forever unless `free_slots_after_run_end_seconds` is set (concurrency-pools warning).
- Per-item fan-out requires ops + `DynamicOut` (or graph-backed assets); plain assets have no per-item retry (dynamic-graphs, dynamic-fanout).
- Automation reactivation can launch thousands of runs (preventing-runs-on-reactivation).
- Partition count guidance 100k; sensors are polling; freshness features moved between `FreshnessPolicy` (deprecated 1.6), freshness checks (superseded 1.12) and new freshness policies (preview, `freshness: enabled: True`).

Scraping pipeline mapping (fetch N pages per source, parse, enrich, publish snapshot): one `StaticPartitionsDefinition(sources)`; assets `raw_pages` (fetch all pages for `context.partition_key`, store bodies via a custom IO manager or write files and return paths), `parsed_events` (same partitions, `deps=[raw_pages]`, `code_version="parser-3"`), `enriched_events` (`pool="llm"` or `pool=f"host:{...}"` cannot be dynamic — pools are static strings), `agenda` (unpartitioned asset depending on all partitions); `@asset_check` for coverage/empty-feed detection with `blocking=True`; `ScheduleDefinition` per partition or `build_schedule_from_partitioned_job`; `run_retries` with `retry_on_asset_or_op_failure: false` for crashes and `RetryPolicy` for HTTP. Plumbing you write: page storage (IO manager or side files), per-page retry/fan-out (dynamic ops inside a graph-backed asset), per-host throttling beyond fixed pools, cursors (sensor cursor string or your own table), coverage metadata as asset metadata, and freeing slots after cancel.

## Known limitations
- "Detecting run worker crashes only works when using a run launcher other than the `DefaultRunLauncher`." (run-monitoring)
- Resume after worker crash only for K8s/Docker launchers+executors (run-monitoring)
- Pool slots leak on cancel/failure by default; deadlock at limit 1 (run-monitoring, concurrency-pools)
- Op retry counter resets per run retry (run-retries)
- `FROM_FAILURE` needs cross-run IO manager (run-retries)
- Executor tag limits are per run only (job-execution)
- `max_catchup_runs` applies to partitioned schedules only; non-partitioned schedules consider only the most recent execution time (`_core/scheduler/scheduler.py`)
- Dequeue order under pools "does **not** guarantee strict FIFO" (run-coordinators)
- Single-run backfills only from asset graph/asset page or jobs sharing the backfill policy (backfilling-data)
- Partition count guidance <= 100,000 (partitioning-assets)
- Freshness policies "not enabled by default while in preview"; anomaly-detection freshness and alerts are Dagster+ (asset-freshness-policies, data-freshness-testing)

## Relevance to Flowyard
- Borrow:
  - Modeling the *product* as assets/snapshots with staleness: `code_version` + data version hashing and an "Unsynced, materialize unsynced" affordance maps directly onto Artifactum action keys and source snapshots.
  - Asset checks as a separate, recorded evaluation (`asset_check_executions` with Planned/Success/Failure) with `blocking` semantics — the right shape for coverage/quality gates on a source snapshot.
  - `run_key` dedup for schedule/sensor-created runs; deterministic dedup identity for scheduled instants.
  - Event log as append-only typed events with `step_key`/`asset_key`/`partition` columns; `STEP_UP_FOR_RETRY`/`STEP_RESTARTED` as explicit records.
  - `retry_on_asset_or_op_failure: false` — distinguishing crash retries from code-failure retries at the run level.
  - `max_catchup_runs` as a small bounded catch-up plus "never before the schedule was turned on".
- Avoid:
  - Requiring a daemon + webserver + code server for basic scheduling.
  - Static pool names as the only resource keying; slot leaks after cancellation.
  - Two overlapping retry systems that compound.
  - Structure-first fan-out (DynamicOut/map/collect) that only works in the op layer, not in the asset layer.
- Answers to open questions:
  - Q1: Not applicable; the plan is fixed per run (snapshot). Subset re-execution just skips steps (`STEP_SKIPPED`); old materializations remain and are marked Unsynced when versions move (asset-versioning-and-caching).
  - Q2: Concurrency pools with `granularity: 'run'` and limit 1, or run-queue `tag_concurrency_limits` on a tag key — both keyed by static strings, persisted in the DB (concurrency-pools, customizing-run-queue-priority).
  - Q3: Daemon. Schedules, sensors, queue and monitoring live in `dagster-daemon`; ad-hoc one-shot execution exists (`dagster job execute`, `execute_in_process`).
  - Q4: Runs are dispatched through a DB-backed queue (`QUEUED` status, run coordinator) to a launcher; inside a run, steps are subprocesses of the run worker (`multiprocess_executor`); pooled steps wait in `pending_steps`.
  - Q5: Persisted (`concurrency_limits`, `concurrency_slots`, `pending_steps`), keyed by pool name; concurrency only — no rate limit/cooldown primitive; slots need explicit cleanup after cancel.
  - Q6: Granularity = asset (persisted object) or op; "Each asset check should test only a single asset property"; within an asset there is no checkpoint, so cheap intermediate work stays inside the function.
  - Q7: Outputs go to IO managers (filesystem pickle under `storage/`, S3, ...); the event log holds metadata only; per-run SQLite shards for events.
  - Q8: Explicit `code_version` string (warn via Unsynced), auto version = run id when absent; never fails; runs snapshot their plan.
  - Q9: Op retry attempts are events in the run's event log (`STEP_UP_FOR_RETRY`, `STEP_RESTARTED`); the delay/jitter is computed in the executor process; run retries create a new run row with retry tags.
  - Q10: Bounded catch-up: `max_catchup_runs=5` for partitioned schedules, only the latest tick otherwise; automation-condition sensors can flood unless time-aware.

## Sources
- https://docs.dagster.io/llms.txt — docs index
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/execution/run-retries.md — run retries, strategies
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/ops/op-retries.md — RetryPolicy, RetryRequested, allow_retries
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/execution/run-coordinators.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/execution/run-monitoring.md — crash detection, timeouts, slot freeing
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/execution/customizing-run-queue-priority.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/execution/dagster-daemon.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/execution/job-timeouts.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/oss/dagster-yaml.md — instance config reference
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/deployment/oss/oss-instance-configuration.md — SQLite file layout
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/assets/defining-assets.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/assets/asset-versioning-and-caching.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/partitions-and-backfills/partitioning-assets.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/partitions-and-backfills/backfilling-data.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/ops/dynamic-graphs.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/examples/best-practices/dynamic-fanout.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/automate/schedules/index.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/automate/schedules/troubleshooting-schedules.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/automate/sensors/index.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/automate/declarative-automation/customizing-automation-conditions/preventing-runs-on-reactivation.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/operate/managing-concurrency/index.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/operate/managing-concurrency/concurrency-pools.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/test/asset-checks.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/test/data-freshness-testing.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/observe/asset-freshness-policies.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/projects/index.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/docs/docs/guides/build/jobs/job-execution.md
- https://raw.githubusercontent.com/dagster-io/dagster/master/examples/docs_snippets/docs_snippets/guides/build/assets/data-assets/data-assets/asset_decorator.py — verbatim example
- https://raw.githubusercontent.com/dagster-io/dagster/master/examples/docs_snippets/docs_snippets/guides/automate/simple-schedule-example.py — verbatim example
- https://raw.githubusercontent.com/dagster-io/dagster/master/examples/docs_snippets/docs_snippets/guides/build/ops_jobs_graphs/retries.py
- https://raw.githubusercontent.com/dagster-io/dagster/master/examples/docs_snippets/docs_snippets/guides/operate/managing_concurrency/pool_concurrency.py
- https://raw.githubusercontent.com/dagster-io/dagster/master/examples/docs_snippets/docs_snippets/guides/build/assets/data-assets/quality-testing/asset-checks/single-asset-check.py
- https://raw.githubusercontent.com/dagster-io/dagster/master/examples/docs_snippets/docs_snippets/guides/automate/sensor-cursor.py
- https://raw.githubusercontent.com/dagster-io/dagster/master/python_modules/dagster/dagster/_core/storage/runs/schema.py
- https://raw.githubusercontent.com/dagster-io/dagster/master/python_modules/dagster/dagster/_core/storage/event_log/schema.py
- https://raw.githubusercontent.com/dagster-io/dagster/master/python_modules/dagster/dagster/_core/storage/schedules/schema.py
- https://raw.githubusercontent.com/dagster-io/dagster/master/python_modules/dagster/dagster/_core/storage/dagster_run.py — status enum
- https://raw.githubusercontent.com/dagster-io/dagster/master/python_modules/dagster/dagster/_core/storage/event_log/sqlite/sqlite_event_log.py — per-run shards
- https://raw.githubusercontent.com/dagster-io/dagster/master/python_modules/dagster/dagster/_core/events/__init__.py — event types
- https://raw.githubusercontent.com/dagster-io/dagster/master/python_modules/dagster/dagster/_core/scheduler/scheduler.py — DEFAULT_MAX_CATCHUP_RUNS
- https://docs.dagster.io/api/clis/cli — CLI command list (via WebFetch summary)
- Could not fetch: `docs/docs/api/clis/cli.md` (404 on GitHub; generated page), `docs_snippets/.../daily_partitioned_asset.py` (404, wrong guess), `fs_io_manager.py` grep for the default output path returned nothing (per-output path layout left unverified).
