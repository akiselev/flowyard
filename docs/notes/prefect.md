# Prefect (3.x)

Researched: 2026-09-16. Scope: Python; docs.prefect.io `v3` pages (markdown served at `*.md`, fetched 2026-09-16, docs mention client 3.6.22+ defaults) and `PrefectHQ/prefect` `main` source (`server/database/orm_models.py`, `task_engine.py`, `_internal/engine.py`, `workers/base.py`).

## Summary
Prefect is a Python workflow library plus an API server (self-hosted `prefect server start` or Prefect Cloud). A flow is an ordinary Python function; tasks are decorated functions that create *task runs* as the flow executes. The server records run/state metadata; task return values are only persisted when result persistence is enabled, and caching is layered on top of persisted results. Prefect optimizes for "add two decorators to existing Python" plus observability, retries, scheduling and concurrency controls; it does not replay orchestration code or checkpoint control flow. It is run by data/ML teams; the same code runs locally (ephemeral or local SQLite server) or via deployments, work pools and workers.

## Deployment and process model
- Client SDK + API server. Self-hosted server: "SQLite (default): Recommended for lightweight, single-server deployments... PostgreSQL: Best for production use, high availability, and multi-server deployments" (concepts/server). Default DB path `~/.prefect/prefect.db` (server-cli). Alembic migrations run automatically at server start; `prefect server database upgrade/downgrade/reset` exist.
- Multi-worker API mode: "PostgreSQL database — SQLite is not supported due to database locking issues" and Redis required for messaging/leases (server-cli).
- A flow run is a Python process. Locally you call `my_flow()`; with deployments a *worker* polls a work pool, submits infrastructure (process/Docker/K8s) and the flow process reports state to the API. Server background services: Scheduler (creates future runs), Late runs marker, cancellation cleanup, automations/events.
- Single-node story: `prefect server start` (SQLite) + run flows in-process or start `prefect worker start` for scheduled deployments. Heartbeats: `PREFECT_FLOWS_HEARTBEAT_FREQUENCY` default 180 s (3.6.22+); zombie handling is an *automation* you must create yourself on self-hosted (advanced/detect-zombie-flows).

## Execution model
- No replay, no checkpointed control flow. The flow function runs top to bottom in a live process; each task call creates a task run via the API; states drive orchestration. Determinism rules: none imposed on user code.
- Recovery after failure/crash is "run the flow function again" (task or flow `retries`, or `prefect flow-run retry`, which "keeps its original ID and parameters, but the `run_count` increments"). Skipping already-done work is entirely the cache's job: "Caching refers to the ability of a task run to enter a `Completed` state and return a predetermined value without actually running the code that defines the task" and the DEFAULT policy hashes "the inputs provided to the task, the code definition of the task, the prevailing flow run ID" — so within the same flow run ID (a retry) identical calls hit the cache, *if* results are persisted.
- Hard precondition (verbatim warning): "Caching requires result persistence, which is off by default." and "any configuration which explicitly avoids result persistence will result in your task never using a cache, for example setting `persist_result=False`."
- Transactions (advanced/transactions): "every Prefect task run is governed by a transaction"; lifecycle BEGIN/STAGE/ROLLBACK/COMMIT; "If a record already exists at the key location the transaction considers itself committed." `commit_mode` LAZY (default; nested tasks stage until the outer block commits), EAGER, OFF. `on_rollback`/`on_commit` hooks on tasks; "rollback hooks must implement cleanup of external side effects... Prefect does not automatically undo those operations."

## Step and operation identity
- Task identity: "Tasks are uniquely identified by a task key, which is a hash composed of the task name and the fully qualified name of the function" (concepts/tasks). Task-run identity inside a flow run is `task_key` + `dynamic_key`, where `dynamic_key_for_task_run` (`_internal/engine.py`) is a per-flow-run counter per task key (`0, 1, 2 ...`), or a `uuid4()` when the task runs detached/remote. Task run names default to `f"{task.name}-{dynamic_key[:3]}"` (`task_engine.py`). So identity is *call order*, not a user-supplied name.
- Cache identity is separate: the cache key (policy or `cache_key_fn(context, parameters)`) names a file in result storage; "Cache keys can be shared by the same task across different flows, and even among different tasks, so long as they all share a common result storage location." `result_storage_key` is a templated filename and "If both `result_storage_key` and `cache_key_fn` are provided, only the `result_storage_key` will be used."
- Mismatch behavior: there is no validation of "same key, different operation". A different key is a cache miss; the same key returns whatever record exists (the docs' `static_cache_key` example returns one value for every input).
- Unreached steps: there is no notion of recorded steps that the new code never reaches. Task runs from earlier attempts stay in the DB (`task_run.flow_run_run_count` distinguishes attempts) and result files stay in storage until `cache_expiration` (none by default). Nothing is flagged or garbage-collected.

## Persistence schema
Source: `src/prefect/server/database/orm_models.py` (main). SQLAlchemy over SQLite (aiosqlite) or Postgres (asyncpg); Alembic migrations in `src/prefect/server/database/_migrations/`.
- `Run` base (shared by `flow_run` and `task_run`): `name`, `state_type` (enum SCHEDULED/PENDING/RUNNING/COMPLETED/FAILED/CANCELLED/CRASHED/PAUSED/CANCELLING), `state_name`, `state_timestamp`, `run_count`, `expected_start_time`, `next_scheduled_start_time`, `start_time`, `end_time`, `total_run_time`, `state_id`.
- `flow_run`: `flow_id`, `deployment_id`, `work_queue_name`, `flow_version`, `deployment_version`, `parameters` (JSON), `idempotency_key`, `context`, `empirical_policy` (FlowRunPolicy: retries, retry delay, pause info), `tags`, `labels`, `infrastructure_pid`, `job_variables`, `parent_task_run_id`, `auto_scheduled`, `work_queue_id`.
- `task_run`: `flow_run_id`, `task_key`, `dynamic_key`, `cache_key`, `cache_expiration`, `task_version`, `flow_run_run_count`, `empirical_policy` (TaskRunPolicy: retries, retry_delay, retry_jitter_factor), `task_inputs` (upstream task-run references), `tags`, `labels`, `state_id`.
- `flow_run_state` / `task_run_state`: one row per transition (type, name, timestamp, message, `state_details`, `data` = result reference); `task_run_state_cache` maps cache keys to states.
- `concurrency_limit` (legacy tag limits: `tag`, `concurrency_limit`, `active_slots` JSON list of task run ids) and `concurrency_limit_v2`: `active`, `name`, `limit`, `active_slots`, `denied_slots`, `slot_decay_per_second`, `avg_slot_occupancy_seconds`.
- `deployment` (`version`, `paused`, `concurrency_limit`, `concurrency_limit_id`, `concurrency_options`, `parameters`, `pull_steps`, `entrypoint`), `deployment_schedule` (`schedule` JSON, `active`, `max_scheduled_runs`, `parameters`, `slug`), `work_pool`, `work_queue` (`concurrency_limit`, `priority`), `log` (`level`, `message`, `flow_run_id`, `task_run_id`), `artifact`, block tables, `configuration`, events/automations tables.
- Results are NOT in the database. "By default, results are persisted to `~/.prefect/storage/`" (or any filesystem block: S3/GCS/Azure). "Result files are JSON documents that contain a `result` field with the serialized data, and a `metadata` field that describes the serializer used"; pickle is the default serializer (base64 of cloudpickle); metadata carries `serializer`, `storage_key`, `expiration`, `prefect_version`. The state row holds a reference. Cache records are collocated with the result file unless `key_storage` is configured. Large outputs are therefore always "referenced", never inlined in the DB; no size limit is documented for self-hosted storage. `cache_result_in_memory=False` drops the value from process memory after commit.

## Crash recovery of in-flight work
- Classification: "`Failed`: The run did not complete because of a code issue and had no remaining retry attempts." vs "`Crashed`: The run did not complete because of an infrastructure issue." and "`CRASHED`: a run in any `CRASHED` state was interrupted by an OS signal such as a `KeyboardInterrupt` or an unexpected `SIGTERM`" (concepts/states). The engine's `handle_crash` sets the state with `force=True` when it can still run.
- If the process dies without reporting, the run stays `Running` until a heartbeat-based automation fires: "Sudden infrastructure failures ... can cause flow runs to become unresponsive and appear stuck in a `Running` state." Self-hosted must create the automation (`EventTrigger(after={"prefect.flow-run.heartbeat"}, expect={"prefect.flow-run.*"}, posture=Proactive, within=timedelta(seconds=90))` -> `ChangeFlowRunState(CRASHED)`); with the 180 s default heartbeat the doc says use `within=540`. Only flows run from deployments emit heartbeats.
- Nothing resumes automatically after `Crashed`. Recovery is `prefect flow-run retry <name|id>` (deployment-backed: back to `Scheduled` for a worker; local: `--entrypoint ./flows/my_flow.py:my_flow`). Work that completed before the crash is skipped only through cache hits (persisted results + matching key); a task that was mid-flight has no record and reruns. There is no user-declared idempotent/retry/manual policy per task for crash cases; `retry_condition_fn` only sees exceptions raised in-process.
- Leases exist only for global concurrency slots ("default lease duration is 5 minutes, ... minimum of 1 minute"; expiry releases the slot). Lease storage default is `prefect.server.concurrency.lease_storage.memory` (settings-ref), i.e. in server memory unless Redis is configured.
- Ephemeral-infra warning: "When a flow run is retried through the UI, a new pod or container is created and cannot access results saved to the local filesystem of the original container."

## Retries, backoff, timeouts
- Declared per task or per flow: `@task(retries=4, retry_delay_seconds=[1, 2, 4, 8])`, `retry_delay_seconds=exponential_backoff(backoff_factor=2)`, `retry_jitter_factor=3`, `retry_condition_fn=retry_handler` where `def retry_handler(task, task_run, state) -> bool` ("If a callable passed to `retry_condition_fn` returns `True`, the task will be retried. Otherwise, the task will exit with an exception."). Globals: `PREFECT_TASKS_DEFAULT_RETRIES`, `PREFECT_TASKS_DEFAULT_RETRY_DELAY_SECONDS="1,10,100"`, `PREFECT_FLOWS_DEFAULT_RETRIES`.
- Mechanics (`task_engine.py` `handle_retry`): the client picks `retry_delay_seconds[min(self.retries, len-1)]` ("repeat final delay value if attempts exceed specified delays") and proposes `AwaitingRetry(scheduled_time=now + delay)`; the attempt counter `self.retries` lives in the engine process; `task_run.run_count` is incremented on every transition into RUNNING and stored server-side; the policy (including `retry_jitter_factor`) is stored in `task_run.empirical_policy`. The `AwaitingRetry` state carries an absolute `scheduled_time`. Where jitter is applied to that time was not verified in the code read.
- Not crash-safe: retries are a loop inside the live process; if it dies during the wait the task run stays `AwaitingRetry`/`Running` and only the heartbeat automation or a manual flow-run retry moves it.
- Failure classification API: `retry_condition_fn`; returning a `Failed(...)` state from a task marks it failed; `return_state=True` lets callers inspect `state.is_failed()`.
- Timeouts: `timeout_seconds` on task/flow -> `TimedOut` (type FAILED). Caveat verbatim: "For sync tasks using `ThreadPoolTaskRunner` (the default), the timeout cannot interrupt blocking operations."; async tasks are cancelled at the next `await`; `ProcessPoolTaskRunner` can interrupt.

## Flow control
- Global concurrency limits (`concurrency_limit_v2`): named slots created via UI/CLI/API; `async with concurrency("database_limit", occupy=1)` holds a slot for the block; `await rate_limit("api_calls", occupy=1)` needs `slot_decay_per_second` on the limit ("Attempting to use `rate_limit` with a limit that has no slot decay will result in an error"). Keyed by a static limit name; stored in the server DB (cross-process, cross-machine); slots are leased (5 min default, renewed by the client; `strict=True` aborts on renewal failure, default logs a warning and continues).
- Tag-based task limits: `PREFECT_TASK_CONCURRENCY`-style limits per tag; since 3.4.19 backed by a global limit named `tag:{tag_name}`; a task with several tags "will run only if *all* tags have available concurrency"; limit 0 "causes immediate abortion"; blocked tasks retry entering Running after 30 s (`PREFECT_SERVER_TASKS_TAG_CONCURRENCY_SLOT_WAIT_SECONDS`).
- Other scopes: work-pool and work-queue flow-run limits, deployment concurrency limit (+ `concurrency_options`), work-queue `priority`. State `AwaitingConcurrencySlot` exists for flow runs waiting on a slot.
- Nothing for debounce, batching, or per-key cooldowns computed at runtime; a "per host" limit is one named limit per host created ahead of time.

## Fan-out, child workflows, map
- `task.map(iterable, static_value)`; iterables are mapped unless wrapped: `sum_plus.map(numbers, unmapped(static_iterable))`. Returns a list of `PrefectFuture`; `futures.result()` "is syntactic sugar for the corresponding list comprehension `[future.result() for future in futures]`" (input order preserved); `wait(futures)`. Runs on the flow's task runner (threads by default; Dask/Ray integrations).
- The work set is not recorded up front: each mapped element becomes a task run as it is submitted; children are identified by the per-flow-run `dynamic_key` counter (or uuid when detached). Failure policy is whatever you write around `.result()`/`return_state=True`; a flow returning an iterable of states fails if any is FAILED.
- Sub-flows: calling a flow inside a flow creates a child flow run (`parent_task_run_id`); `run_deployment` creates a separately cancellable run.

## Versioning and code change policy
- Flow `version`: "If not provided, we will attempt to create a version string as a hash of the file containing the wrapped function"; task `version` free string; both stored on runs (`flow_version`, `deployment_version`, `task_version`).
- Cache invalidation on code change is opt-in via `TASK_SOURCE`, which "only considers raw lines of code in the task (and not the source code of nested tasks)". No compatibility gate: in-flight runs keep running old code; retries load whatever code the worker pulls now. No warn/fail on mismatch, no migration tooling.

## Schedules, timers, missed runs
- Schedule types cron/interval/rrule on deployments (`deployment_schedule` rows; `active`, `max_scheduled_runs`, per-schedule `parameters`). The Scheduler service (loop 60 s) "creates the fewest runs that satisfy": "No more than 100 runs will be scheduled", "not ... more than 100 days in the future", "At least three runs", "until at least one hour in the future". Runs are ordinary `flow_run` rows in `Scheduled` with `expected_start_time`.
- Late: "`Late`: The run's scheduled start time has passed, but it has not transitioned to PENDING (15 seconds by default)" (`PREFECT_SERVER_SERVICES_LATE_RUNS_AFTER_SECONDS=PT15S`, loop 5 s).
- Missed runs after downtime: the worker query has no lower bound: `scheduled_before = now + prefetch_seconds` (`workers/base.py::_get_scheduled_flow_runs`), so every Late run is picked up. Issue #7006 describes exactly this ("Your agent went down yesterday at 9 AM ... restarted ... the following day" and all missed hourly runs execute) and asks for bulk clearing; #17575 asks for "a way to skip the late runs if they get late beyond a threshold". No catch-up cap or "one current run" policy exists in docs. Mitigation: "If you change a schedule, previously scheduled flow runs that have not started are removed", pause the schedule, or deployment concurrency limits.
- Durable timers: `AwaitingRetry`/`Scheduled` states carry absolute times; `pause_flow_run`/`suspend_flow_run` exist (Suspended: "the process has exited"). No general durable sleep inside a task.

## Cancellation and late results
- Request: set `CANCELLING`; "the worker ... sends a signal to the flow run infrastructure ... If the run does not terminate after a grace period (default of 30 seconds), the infrastructure is killed". Enforcement uses `infrastructure_pid` (hostname+PID / container id / job name). Verbatim: "Flow run cancellation requires that the flow run is associated with a deployment." and "Inline nested flow runs ... cannot be cancelled without cancelling the parent flow run." Server safety net: `CANCELLING` longer than 300 s -> `CANCELLED`.
- Late results after cancellation: state transitions are validated server-side (the engine sometimes uses `force=True`); what happens to a task completing after cancel is not documented in the pages read — unverified.

## Observability and tooling
- UI at `http://127.0.0.1:4200` (runs, task runs, states, logs, artifacts, concurrency limits, automations).
- CLI: `prefect flow-run ls --state FAILED`, `prefect flow-run retry <name|uuid> [--entrypoint path:flow]`, `prefect flow-run cancel <id>`, `prefect deployment schedule ...`, `prefect server start [--workers N]`, `prefect server database upgrade|downgrade|reset`, `prefect config view --show-defaults`, `prefect block ls`, `prefect experimental result-storage set|inspect|clear` (beta). MCP server available.
- Cache explanation: only via state names (`Cached`) and reading result files with `ResultStore(result_storage=...).read(key)`; no "explain cache" command. Logs are DB rows (`log` table) plus AI log summaries (Cloud).

## API ergonomics
Minimal complete example, verbatim from https://docs.prefect.io/v3/examples/simple-web-scraper (Python; no Rust SDK):
```python
from __future__ import annotations
import requests
from bs4 import BeautifulSoup
from prefect import flow, task

@task(retries=3, retry_delay_seconds=2)
def fetch_html(url: str) -> str:
    """Download page HTML (with retries).
    This is just a regular requests call - Prefect adds retry logic
    without changing how we write the code."""
    print(f"Fetching {url} …")
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.text

@task
def parse_article(html: str) -> str:
    """Extract article text, skipping code blocks.
    Regular BeautifulSoup parsing with standard Python string operations.
    Prefect adds observability without changing the logic."""
    soup = BeautifulSoup(html, "html.parser")
    # Find main content - just regular BeautifulSoup
    article = soup.find("article") or soup.find("main")
    if not article:
        return ""
    # Standard Python all the way
    for code in article.find_all(["pre", "code"]):
        code.decompose()
    content = []
    for elem in article.find_all(["h1", "h2", "h3", "p", "ul", "ol", "li"]):
        text = elem.get_text().strip()
        if not text:
            continue
        if elem.name.startswith("h"):
            content.extend(["\n" + "=" * 80, text.upper(), "=" * 80 + "\n"])
        else:
            content.extend([text, ""])
    return "\n".join(content)

@flow(log_prints=True)
def scrape(urls: list[str] | None = None) -> None:
    """Scrape and print article content from URLs.
    A regular Python function that composes our tasks together.
    Prefect adds logging and dependency management automatically."""
    if urls:
        for url in urls:
            content = parse_article(fetch_html(url))
            print(content if content else "No article content found.")

if __name__ == "__main__":
    urls = [
        "https://www.prefect.io/blog/airflow-to-prefect-why-modern-teams-choose-prefect"
    ]
    scrape(urls=urls)
```
(Code lines as published across the page's four code blocks; blank lines between statements omitted.)

Delightful:
- Two decorators; retries, jitter and a retry predicate are keyword arguments; no IDs or registration.
- Cache policies compose with `+` and `-` (`(TASK_SOURCE + INPUTS).configure(key_storage=..., isolation_level=SERIALIZABLE, lock_manager=...)`, `INPUTS - 'debug'`).
- `transaction(key=...)` with `txn.is_committed()` and `@task.on_rollback` gives keyed idempotency and compensation in a few lines.
- `retry_condition_fn(task, task_run, state)` sees the exception via `state.result()` and can inspect HTTP status codes.
- Zero-config local server on SQLite; `.map()` + `unmapped()`; `return_state=True`.

Painful / footguns (each with the doc that shows it):
- Caching silently does nothing until `PREFECT_RESULTS_PERSIST_BY_DEFAULT=true` (concepts/caching warning); DEFAULT cache keys include the flow run ID, so cross-run reuse needs an explicit policy.
- `TASK_SOURCE` ignores nested tasks/helpers; `result_storage_key` silently overrides `cache_key_fn` (advanced/results warning).
- READ_COMMITTED (default) "allows multiple executions of the same task to occur simultaneously"; SERIALIZABLE needs a lock manager per execution context (memory/file/Redis) (advanced/caching).
- Sync task timeouts cannot interrupt blocking code with the default thread runner (write-and-run).
- Cancellation needs a deployment and a worker; zombie detection needs a hand-built automation on self-hosted.
- Late runs pile up after downtime (#7006, #17575). SQLite "database is locked" under concurrency (#10188, #10956); multi-worker server refuses SQLite.
- Results are pickled by default; deserialization requires the same classes/serializers importable ("Serializer ... is not available in this environment").
- `rate_limit` errors unless the limit was created server-side with slot decay; limit names are static strings, so per-host limits must be pre-created.

Scraping pipeline (fetch N pages per source, parse, enrich, publish snapshot): `@flow def refresh(source)` calls `pages = fetch_page.map(urls)` (task with `retries`, `retry_condition_fn`, and `with concurrency(f"host:{host}")` inside), `parsed = parse.map(pages)` with `cache_policy=INPUTS + TASK_SOURCE` and persisted results, `enrich.map(parsed)`, then `with transaction(key=f"publish:{source}:{snapshot_id}")` around `publish(...)`. Plumbing you write yourself: create every concurrency/rate limit ahead of time (names are static); decide and enable result persistence/storage for page bodies (each task run writes a pickle file; no artifact store or dedup); per-source cursors/coverage metadata (no state primitive); zombie automation; a scheme to avoid Late-run bursts; per-item failure collection around futures.

## Known limitations
- "Caching requires result persistence, which is off by default." (concepts/caching)
- "`TASK_SOURCE` ... only considers raw lines of code in the task (and not the source code of nested tasks)" (concepts/caching)
- READ_COMMITTED allows concurrent duplicate execution; SERIALIZABLE requires `lock_manager` (advanced/caching)
- "Flow run cancellation requires that the flow run is associated with a deployment." (advanced/cancel-workflows)
- Zombie detection managed automation "is only available in Prefect Cloud"; self-hosted builds it manually (advanced/detect-zombie-flows)
- Multi-worker server: "SQLite is not supported due to database locking issues" (server-cli); `sqlite3.OperationalError: database is locked` reports: #10188, #10956, #7277
- Missed-run pile-up after worker downtime: #7006 (closed feature request), #17575 (closed enhancement)
- Thread-runner timeouts cannot interrupt blocking sync code (write-and-run)
- Ephemeral infrastructure loses local results across UI retries (advanced/results warning)
- RRule: `COUNT` unsupported, 6,500-char max (concepts/schedules)

## Relevance to Flowyard
- Borrow:
  - Cache-policy algebra as *policy objects* (INPUTS / TASK_SOURCE / RUN_ID composable, parameter subtraction, `key_storage` separate from result storage) — a good shape for Artifactum's action-key policy.
  - `retry_condition_fn(task, run, state) -> bool` plus `retry_jitter_factor` and list-valued delays; `Late` as an explicit, queryable state; `run_count` on the run row and `flow_run_run_count` on child rows.
  - Result record metadata (`serializer`, `storage_key`, `expiration`, engine version) stored with every persisted output.
  - Transactions: keyed `is_committed()` early exit and `on_rollback` hooks; LAZY vs EAGER commit vocabulary.
  - Slot-decay rate limits and leased slots as the model for named cooldowns; the "limit must exist before use" error is a good fail-fast.
- Avoid:
  - Two separate "persist" and "cache" switches; recovery that depends on an opt-in cache.
  - Identity by call-order counter (`dynamic_key`); no validation when a key maps to a different operation.
  - Unbounded Late-run catch-up (no lower bound in the worker query).
  - Leases/heartbeat detection as the only crash story; in-memory lease storage by default.
  - Pickle-by-default result files keyed by hash with no owner/retention metadata in the DB.
- Answers to open questions:
  - Q1: Not modeled. Prefect has no replay; old task runs and result files simply remain (concepts/caching, orm_models `task_run.flow_run_run_count`).
  - Q2: Yes, per deployment (`deployment.concurrency_limit` / `concurrency_options`, "Prevent concurrent runs of a specific deployment") and per static global-limit name with `occupy=1`; not per parameter value.
  - Q3: Both. Flow runs are one-shot processes; `prefect server` (scheduler, late-runs services) and `prefect worker` are long-running daemons.
  - Q4: Inline. Tasks execute in the flow process via the task runner (threads by default, `ProcessPoolTaskRunner` optional); only flow runs are queued (Scheduled -> worker pull) (concepts/task-runners, workers/base.py).
  - Q5: Persisted server-side in `concurrency_limit_v2` keyed by name; rate limits via `slot_decay_per_second`; slot leases default in-memory (`server.concurrency.lease_storage`).
  - Q6: No numeric guidance; the scraper example separates network IO from parsing "so both pieces can be retried or cached independently"; each task run costs API round trips, state rows and (if persisted) a file.
  - Q7: Never inline in the DB; results go to result storage files (local dir or object store) with JSON+base64 pickle; no documented size cap for self-hosted.
  - Q8: Warn-and-continue is not even present: no gate at all. Optional `TASK_SOURCE` hashing (excludes nested code) and free-form `version` strings.
  - Q9: `run_count` is persisted; `AwaitingRetry` carries an absolute `scheduled_time`; the attempt counter and delay selection are client-side in the live process (`task_engine.py::handle_retry`), so a crash mid-wait is not resumed.
  - Q10: Catch-up burst: all Late runs execute when a worker returns (`scheduled_before` has no lower bound; #7006). Only manual mitigations.

## Sources
- https://docs.prefect.io/llms.txt — docs index
- https://docs.prefect.io/v3/concepts/caching.md — cache policies, persistence warning, isolation levels
- https://docs.prefect.io/v3/advanced/caching.md — key_storage, SERIALIZABLE example, multi-task cache in transaction
- https://docs.prefect.io/v3/advanced/results.md — result persistence, storage, file format, ResultStore
- https://docs.prefect.io/v3/advanced/transactions.md — transaction lifecycle, commit modes, idempotency key, lock managers
- https://docs.prefect.io/v3/how-to-guides/workflows/retries.md — retries, delays, jitter, retry_condition_fn
- https://docs.prefect.io/v3/concepts/states.md — state table, Failed vs Crashed
- https://docs.prefect.io/v3/advanced/detect-zombie-flows.md — heartbeats and automation
- https://docs.prefect.io/v3/concepts/schedules.md — schedule types, Scheduler service constraints
- https://docs.prefect.io/v3/concepts/server.md — databases, migrations
- https://docs.prefect.io/v3/how-to-guides/self-hosted/server-cli.md — SQLite default path, DB commands, multi-worker requirements
- https://docs.prefect.io/v3/concepts/global-concurrency-limits.md — slots, leases, slot decay
- https://docs.prefect.io/v3/concepts/tag-based-concurrency-limits.md — tag limits
- https://docs.prefect.io/v3/how-to-guides/workflows/run-work-concurrently.md — .map, unmapped, futures
- https://docs.prefect.io/v3/how-to-guides/workflows/retry-flow-runs.md — manual retry CLI
- https://docs.prefect.io/v3/advanced/cancel-workflows.md — cancellation
- https://docs.prefect.io/v3/concepts/tasks.md — task key definition
- https://docs.prefect.io/v3/how-to-guides/workflows/write-and-run.md — timeouts, version
- https://docs.prefect.io/v3/examples/simple-web-scraper.md — verbatim example
- https://docs.prefect.io/v3/api-ref/settings-ref.md — heartbeat, late_runs, lease_storage, prefetch settings
- https://raw.githubusercontent.com/PrefectHQ/prefect/main/src/prefect/server/database/orm_models.py — schema
- https://raw.githubusercontent.com/PrefectHQ/prefect/main/src/prefect/task_engine.py — retry/crash handling
- https://raw.githubusercontent.com/PrefectHQ/prefect/main/src/prefect/_internal/engine.py — dynamic_key_for_task_run
- https://raw.githubusercontent.com/PrefectHQ/prefect/main/src/prefect/workers/base.py — scheduled-run query
- https://github.com/PrefectHQ/prefect/issues/7006 — missed runs executed on agent restart
- https://github.com/PrefectHQ/prefect/issues/17575 — skip late runs beyond threshold
- https://github.com/PrefectHQ/prefect/issues/10188, /10956, /7277 — SQLite "database is locked" (found via search; not fetched individually)
- Could not fetch: none of the above failed; WebFetch of `/v3/concepts/results`, `/v3/concepts/transactions`, `/v3/how-to-guides/automations/detect-zombie-flows` redirected to the intro page (wrong paths; replaced by the `.md` URLs above).
