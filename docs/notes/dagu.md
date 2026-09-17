# Dagu

Researched: 2026-09-16. Scope: Go single binary; repository `github.com/dagu-org/dagu` (module now `github.com/dagucloud/dagu/v2`), shallow clone at commit `b1a5616a3d16` (2026-09-16), reading `README.md`, `llms.txt`, `ARCHITECTURE.md`, `LICENSING.md`, `schemas/dag.schema.json`, `internal/{ir,runtime,persis,queue,schedulerstate,service/scheduler,cmd}`; docs.dagu.sh pages "Durable Execution" and "Scheduling" via a summarizing fetch. Field names below use the current snake_case YAML (`retry_policy`, `max_active_runs`); older docs use camelCase.

## Summary
Dagu is a self-contained workflow engine: "It runs as a single binary without requiring an external database or message broker. It stores state locally by default". Workflows are YAML DAGs of shell/action steps (`type: graph|chain|agent|build`); the same binary is the CLI, HTTP server/UI, cron scheduler, coordinator and worker. It optimizes for operating scheduled command pipelines on one machine (or a few workers) with minimal setup: file-backed run history, retries, catch-up, queues, human tasks, artifacts. It is GPL-3.0-or-later with paid self-host add-ons (SSO/RBAC/audit) and a fast-moving feature set (LLM "agent" DAGs, remote actions, build reuse).

## Deployment and process model
- One binary, roles chosen at start: "**Server** serves the Web UI and REST API. **Scheduler** owns `schedule:` and drains the queue. **Coordinator** is the gRPC endpoint workers poll ... **Worker** polls a coordinator ... `dagu start-all` runs the server, scheduler, and coordinator in one process." Topologies: single server, temporary workers on a shared volume, distributed workers over gRPC.
- One-shot execution: `dagu start hello.yaml` runs the DAG in the calling process. Scheduled and queued runs are separate `dagu start` processes: scheduler code comments say "Non-catchup runs are dispatched in a goroutine (process spawn can be slow)" and catch-up runs are enqueued ("enqueue is fast file I/O").
- Single-server picture (README "Architecture"):
```sh
  ┌─────────────────────────────────────────┐
  │  dagu start-all                         │
  │  ┌───────────┐ ┌───────────┐ ┌────────┐ │
  │  │ HTTP / UI │ │ Scheduler │ │Executor│ │
  │  └───────────┘ └───────────┘ └────────┘ │
  │  File-based storage (logs, state, queue)│
  └─────────────────────────────────────────┘
```
- What must be running: nothing for `dagu start`; the scheduler for `schedule:`, queues, catch-up, zombie detection and DAG-level auto-retries ("Requirements for DAG-level retry: Scheduler must be running"); the server only for UI/API.
- Paths (README): `DAGU_DAGS_DIR` `~/.config/dagu/dags`, `DAGU_DATA_DIR` `~/.local/share/dagu/data`, `DAGU_LOG_DIR` `~/.local/share/dagu/logs`, `DAGU_DAG_STATE_DIR` `{data}/dag-state`, `DAGU_DAG_RUN_WORK_DIR` `{data}/dag-run-work`; `DAGU_HOME` overrides all. Scheduler env: `DAGU_SCHEDULER_ZOMBIE_DETECTION_INTERVAL` 45s, `DAGU_SCHEDULER_LOCK_STALE_THRESHOLD` 30s, `DAGU_QUEUE_ENABLED` true.

## Execution model
- Explicit persisted structure: YAML -> `internal/spec` -> IR (`ir.DAG`, `ir.Step`) -> `runtime.Plan` of nodes with dependency edges; the executor runs commands/actions and appends status snapshots. Verbatim (`dagrun.go`): "`status.jsonl` ... contains the status of the dag-run in JSON Lines format. While running the dag-run, new lines are appended to this file on each status update. After finishing the run, this file will be compacted into a single JSON line file."
- No orchestration code to replay; resumption is "build a new attempt from the last persisted node statuses". `CreateRetryPlan` -> `setupRetry` "resets the state of failed/aborted nodes and their dependents": a node is cleared when its status is `NodeFailed`, `NodeRetrying`, `NodeAborted` or `NodeRejected`, or when any dependency was cleared; succeeded/skipped nodes keep their recorded status and outputs.
- Determinism rules: none; steps are shell commands. Outputs are captured to variables (`output:`), declared `outputs:` via `$DAGU_OUTPUT_FILE` "captured only after the command succeeds", and artifacts.
- Run statuses (`ir.Status`): `not_started, running, failed, aborted, succeeded, queued, partially_succeeded, waiting, rejected`; node statuses (numeric, "persisted in run status snapshots"): `NotStarted=0, Running=1, Failed=2, Aborted=3, Succeeded=4, Skipped=5, PartiallySucceeded=6, Waiting=7, Rejected=8, Retrying=9`.

## Step and operation identity
- Steps are identified by `name` ("Unique identifier for the step within this DAG. If omitted, Dagu will automatically generate a name") and optional `id` ("Must be unique within the DAG"; used in `${id.stdout}`, `${steps.<id>.outputs.<name>}`). Nodes in `status.jsonl` embed the full `step` definition plus `status`, `retryCount`, `doneCount`, `retriedAt`, `skippedByRetry`, `stdout`/`stderr` log paths, `outputVariables`, `children` (sub-DAG runs).
- Retry rebinds recorded nodes to the *current* DAG's steps by name (`rebindRetryNodesToSteps`, `retryStepForNode`) and errors when a recorded step no longer exists (`ErrMissingNode` for the target step; exact behavior for other missing steps not read — partially verified). Extra steps in new YAML are not part of an old attempt's node list (unverified beyond the plan code).
- Unreached steps: a skipped-by-preconditions node is `Skipped`; `continue_on.skipped` decides whether the DAG continues. Nothing else is recorded for "code paths not taken".
- Scheduled and catch-up runs get deterministic IDs: `GenerateCatchupRunID(dagName, scheduledTime)` = `"catchup-"` + hash, and enqueue checks "Catchup run already exists; skipping".

## Persistence schema
File-backed (`internal/persis/file`, `internal/persis/store`); no database.
- Run history: `<dag-runs base>/<safe dag name>/dag-runs/YYYY/MM/DD/dag-run_<ts>_<runID>/a_<YYYYMMDD_HHMMSS_mmmZ>_<attemptID>/status.jsonl` (constants `DAGRunDirPrefix = "dag-run_"`, `AttemptDirPrefix = "a_"`, legacy `attempt_`; `attemptDirName(ts, id)`), plus the DAG definition file (`DAGDefinition`), an outputs file (`OutputsFile`), a `messages/` dir for LLM steps, a `CancelRequestedFlag` file, and `sub/` for sub-DAG runs (legacy `children/`). A per-day index (`dagrunindex`) caches summaries "without reading status.jsonl". Directory-level locks (`dirlock`, stale threshold 30 s).
- Resulting tree (assembled from the constants above; timestamps abbreviated):
```text
<data>/dag-runs/<dag>/dag-runs/2026/09/16/
  dag-run_20260916_120000Z_<runID>/
    a_20260916_120000_000Z_<attemptID>/
      status.jsonl            # appended per update, compacted on finish
      <dag definition file>   # copy of the YAML used by this attempt
      <outputs file>          # collected step outputs
      messages/               # per-step LLM messages (agent DAGs)
      cancel-requested flag   # written by `dagu stop`
    sub/dag-run_<ts>_<subRunID>/a_<ts>_<attemptID>/status.jsonl
<data>/proc/<group>/proc_<ts>_<runid>_<attemptid>.proc   # heartbeat
<data>/queue/... <itemID>.json                          # queued run refs (+index)
<data>/dag-state/...                                    # state.* JSON
<logs>/...<step>.log                                    # per-node stdout/stderr paths
```
- `DAGRunStatus` fields (json): `root`, `parent`, `name`, `dagRunId`, `attemptId`, `attemptKey`, `claimKey`, `status`, `conditions`, `triggerType`, `workerId`, `pid`, `pidStartedAt`, `nodes[]`, `onInit/onExit/onSuccess/onFailure/onAbort/onWait`, `createdAt`, `queuedAt`, `scheduleTime`, `startedAt`, `finishedAt`, `autoRetryCount`, `autoRetryLimit`, `autoRetryInterval`, `autoRetryBackoff`, `autoRetryMaxInterval`, `procGroup`, `definitionId`, `log`, `workingDir`.
- Process liveness: `proc_<ts>_<runid>_<attemptid>.proc` files with JSON meta (`dag_name`, `dag_run_id`, `attempt_id`, `root_*`, `started_at`) and an 8-byte heartbeat, written every 5 s, stale after 90 s (`proc/store.go` defaults).
- Queue: `internal/persis/store/queue.go` stores one record per queued item (`FileName: itemID + ".json"`, `DAGRun`, `QueuedAt`) in a collection per queue name with priority high/low and an index; the run itself is created first with status `queued`.
- Scheduler state (`schedulerstate.State`): per-DAG watermark `lastScheduledTime`, `startScheduleFingerprint`, `skipSuccessResetAt`, `oneOffs{scheduledTime,status}`, `nextRun`.
- Step outputs: stdout/stderr go to log files (`Stdout`/`Stderr` paths recorded per node); captured `output:` values are inlined in the status snapshot and capped: `max_output_size` "Defaults to 1MB (1048576 bytes)" — exceeding it fails the step. Large data goes to artifacts (`stdout.artifact`, `artifact.*` actions, `DAG_RUN_ARTIFACTS_DIR`) or `type: build` path outputs. Small cross-run state: `state.get/set/...` JSON under `dag-state`.
- Retention: `hist_retention_days` or `hist_retention_runs` ("Mutually exclusive"); `dagu rm --history` / deprecated `dagu cleanup <dag> --retention-days` ("Active runs are never deleted").

## Crash recovery of in-flight work
- Detection: the scheduler's `ZombieDetector` (interval 45 s, `0` disables) lists proc entries, counts consecutive stale observations per attempt, and only when the persisted attempt is still `Running` and local (`WorkerID == "" || "local"`) — and after checking that the recorded PID identity is not still alive ("Skipping zombie repair because local process identity is still alive despite stale proc heartbeat") — calls `runtime.RepairStaleLocalRun`, which rebuilds missing nodes from the DAG if needed and `markActiveStatusFailed(...)` writes the attempt as `failed` (with a stale-run error) and removes the proc file. Remote/worker runs are left to the coordinator.
- No automatic resume of the interrupted attempt. Options: (a) manual `dagu retry --run-id <id> <dag>` (new attempt, same run ID; failed/aborted nodes and their dependents rerun; succeeded nodes are kept); (b) DAG-level `retry_policy` (root): "The scheduler scans recent failed DAG runs and queues another attempt when the retry delay has elapsed. This retry path creates a new attempt under the same DAG-run ID." with `autoRetryCount/Limit/...` persisted in the status and "Only scheduler-issued DAG retries consume this retry budget. Manual retry does not increment the DAG auto-retry counter."
- What the user can declare per step about an interrupted command: nothing (no idempotent/uncertain classification). A step that was `Running` at the crash is marked failed by the repair and will simply rerun on retry.
- `type: build` steps add a recovery contract: "Each retry receives a new staging path. A failed, timed-out, or aborted attempt removes its staging file and leaves the previous final output and manifest unchanged." and "Crash recovery refuses to overwrite a final output that was externally replaced".

## Retries, backoff, timeouts
- Step: `retry_policy: {limit, interval_sec, backoff (multiplier), max_interval_sec, exit_code: [..]}`; "If not specified, all non-zero exit codes will trigger a retry" (`RetryPolicy.ShouldRetry`). Step retries "run inside the current DAG attempt. They rerun the same step. They do not create a new DAG-run ID and they do not create a new DAG attempt." Persisted per node: `retryCount`, `retriedAt`, status `retrying`; the evaluated policy is written back into the step snapshot ("Persist the evaluated retry policy so status snapshots carry the concrete values"). The wait itself is in-process; a crash during the wait becomes a zombie-repaired failure. No jitter field.
- Manual step retry: `dagu retry --run-id X --step build [--downstream]`; "manual step retries start with a fresh retry budget" unless the node was mid-retry.
- DAG-level: root `retry_policy: {limit, interval_sec, backoff, max_interval_sec}` handled by the scheduler `RetryScanner` ("periodically discovers failed latest attempts and enqueues DAG-level retries once their backoff has elapsed"); needs `scheduler.retry_failure_window > 0`, latest attempt, top-level DAG.
- Preconditions (DAG and step level): each entry sets "exactly one of `condition` or `eval`"; `condition` is "command text to execute when `expected` is omitted" or a value to compare; `expected` "Supports regex patterns with 're:' prefix"; `negate` inverts. A failed precondition yields `Skipped` (see `continue_on.skipped`); DAG-level `preconditions` gate the whole run. The DAG-level `steps` defaults block can set `preconditions` that are "additive with step-level preconditions".
- `repeat_policy: {repeat: true|while|until, interval_sec, limit, backoff, max_interval_sec, condition, expected, exit_code}` for polling loops. `continue_on: {failure, skipped, exit_code, output, mark_success}` classifies acceptable failures. Timeouts: DAG `timeout_sec`, step `timeout_sec` ("takes precedence over the DAG-level timeout"), `delay_sec` before first node.

## Flow control
- Queues: DAG `queue:` name ("If not specified, defaults to the DAG name"); global `queues.config[{name, max_concurrency}]` in `config.yaml`; `disable_queue: true` runs immediately; `max_active_runs` is "DEPRECATED: This field is ignored for local (DAG-based) queues". Queue is file-backed, persistent, cross-process (scheduler drains it), with high/low priority (`dagu enqueue`, `dagu dequeue` "marks it as aborted").
- Within a run: `max_active_steps` ("Non-positive values mean unlimited"), `parallel.max_concurrent` ("default: 10, maximum: 1000"), `foreach.max_concurrent` default 10. Schedule-level: `overlap_policy: skip|all|latest` (default `skip`), `skip_if_successful`.
- No rate limiter, cooldown or per-host keying; a host limit is a named queue or a child DAG with its own queue.

## Fan-out, child workflows, map
- `action: dag.run` runs a child DAG ("Sub-DAGs do not inherit parent env vars; pass what you need via `params:`"). `parallel:` "currently requires `action: dag.run`" with static `items:` or `parallel: ${params.ITEMS}`; "Each child invocation receives the current item as `ITEM`." Child runs are persisted as sub DAG runs (`children` on the node, `sub/` directory) with their own run IDs, retryable via `--sub-run-id`. Results: `${step_id.outputs.*}` and object-form outputs; the README example shows `max_concurrent: 2`.
- README "Parallel Sub-DAG execution" example, verbatim:
```yaml
steps:
  - id: patch
    action: dag.run
    with:
      dag: patch-host
      params:
        host: ${ITEM}
    parallel:
      items:
        - web-1.internal
        - web-2.internal
        - db-1.internal
      max_concurrent: 2

---

name: patch-host
params:
  - name: host
    type: string
ssh:
  user: deploy
  host: ${params.host}
steps:
  - id: apply
    run: apt-get update -q && apt-get upgrade -y
```
- `foreach:` inline body: `items`, `as` (default `item`), `key` ("Optional expression producing a stable key for each item"), `max_concurrent`, `steps`, `collect` (output names mapped to expressions).
- The item set is captured when the step starts (from params/outputs); failure policy of the collection is via `continue_on` on the parent step; ordering of collected outputs not documented (unverified).

## Versioning and code change policy
- Each attempt stores its DAG definition (`DAGDefinition` file); retry rebinds nodes to steps by name against the reloaded DAG ("The DAG is reloaded from source before persistence so queued catchup retries" pick up current YAML). No version attribute, no compatibility check beyond missing-step errors.
- Build DAGs hash "the resolved recipe, declared input contents, and current output"; a mismatch produces an `execute` decision, `--no-reuse` forces recompute; `dagu dry` previews decisions. Operational warning: "do not let old and new Dagu versions execute the same run concurrently".
- `dagu validate`, `dagu schema dag steps.retry_policy` expose the schema; `startScheduleFingerprint` in scheduler state detects schedule edits.

## Schedules, timers, missed runs
- `schedule:` cron list (with `start`/`stop`/`restart` forms and timezone), one-off `at:` RFC 3339 entries ("Dagu runs that timestamp once and then marks it consumed"). Scheduler ticks per minute (`NextTick` truncates to the minute) and persists the per-DAG watermark.
- Missed runs: `catchup_window` "Lookback horizon for replaying missed cron runs when the scheduler restarts ... If omitted, missed runs are not replayed." Replay start = `max(now - catchupWindow, lastTick, lastScheduledTime)` (`ComputeReplayFrom`); `MaxMissedRuns = 1000` "only the most recent runs are kept"; missed ticks are enqueued chronologically with deterministic IDs and require queues enabled. `overlap_policy` governs a ready catch-up run while the DAG is still running: `skip` "drops the catchup run and moves to the next", `all` "keeps it in the buffer and retries on the next scheduler tick", `latest` "discards all but the most recent missed interval". `skip_if_successful`: "checks if this DAG has already succeeded since the last scheduled time ... Manual triggers always run regardless".
- Timers: `delay_sec`, `repeat_policy.interval_sec`, `timeout_sec`; no durable sleep beyond the persisted schedule watermark and queue.
- HA: scheduler lock with stale detection (30 s).

## Cancellation and late results
- `dagu stop <dag> [--run-id]` / API: a `CancelRequestedFlag` file in the attempt dir; per-step `signal_on_stop` ("If empty, uses same signal as parent process"); status `aborted`; `handler_on.abort/exit` hooks. `dagu dequeue` aborts queued runs. Retry treats `aborted` nodes like failed ones.
- Late completion after abort: not documented; the attempt writer is the single process, so a killed process cannot write late results, but a step that finishes after the flag is set is not discussed (unverified).

## Observability and tooling
- Web UI on :8080 (DAG list/editor, run history, per-step logs, artifacts, retry, edit-retry preview, reschedule, dequeue). REST API v1 paths include `/dag-runs/{name}/{dagRunId}`, `/spec`, `/reschedule`, `/edit-retry/preview`, `/edit-retry`, `/sub-dag-runs`, `/dequeue`, `/log`, `/log/download`, `/artifacts` (`api/v1/api.yaml`). MCP server, SSE, Prometheus metrics, structured logs, notifications.
- CLI: `dagu start|enqueue|exec|dry|validate|status|history|ls|ps|rm|cleanup|schema|config|stop|restart|retry|dequeue|human-task complete|server|scheduler|start-all|coordinator|worker` (31 commands). `dagu history --status failed --last 7d --format json|csv`; `dagu status <dag> --run-id --sub-run-id`; `dagu ps` lists live proc entries; `dagu retry --run-id <id> [--step <name>] [--downstream] [--sub-run-id <id>]`.
- `dagu history` defaults: "shows runs from the last 30 days, newest first", `--limit` max 1000, `--run-id` partial match; statuses filterable: `running, succeeded, failed, aborted, queued, waiting, rejected, not_started, partially_succeeded`.
- Cache explanation: `dagu dry` shows `reuse`/`execute` decisions for build steps; node `build` field records the decision.

## API ergonomics
Minimal complete example, verbatim from https://github.com/dagu-org/dagu/blob/main/README.md ("Create and run a workflow"):
```yaml
steps:
  - id: hello
    run: echo "hello from Dagu"
```
```sh
dagu start hello.yaml
```
Retry/parallel excerpts from the same README:
```yaml
steps:
  - name: flaky-api-call
    run: curl -f https://api.example.com/data
    retry_policy:
      limit: 3
      interval_sec: 10
    continue_on:
      failure: true
```
```yaml
schedule:
  - "0 */6 * * *"          # Every 6 hours
overlap_policy: skip       # Skip if previous run is still active
catchup_window: "5h"       # Catch up missed runs when scheduler is down for up to 5 hours
```
Delightful:
- Zero infrastructure; one YAML file and one command; UI/scheduler/queue in the same binary.
- Retry-from-step with `--downstream`, retries reusing the run ID as new attempts, deterministic catch-up run IDs, `overlap_policy`/`catchup_window`/`skip_if_successful` as three explicit knobs.
- `dagu schema dag <path>` and `dagu validate` remove guessing; `dagu dry` previews.
- `type: build` gives content-hashed reuse with atomic publication and explicit refusal to clobber externally changed outputs.
- `continue_on`, `exit_code` filters and `repeat_policy` cover most shell-level failure semantics.

Painful / footguns (cites):
- It is YAML + shell, not a typed function API; values are strings ("`params:` values arrive as strings"), `${step_id.stdout}` "is a log file path, not stdout content", sub-DAGs "do not inherit parent env vars" (llms.txt).
- Fan-out only through child DAGs: "`parallel:` currently requires `action: dag.run`"; build reuse is local-only ("Distributed execution is rejected"); human tasks "cannot be used in sub-DAGs" (llms.txt).
- 1 MB output cap fails the step (`max_output_size`); large data must go through artifacts.
- No jitter; per-step retry wait is in-process; DAG-level auto-retry needs the scheduler running and a `retry_failure_window`.
- Crash handling is detection + mark failed (zombie detector, 45 s interval, 90 s stale) rather than resume; missed-run replay is silently off unless `catchup_window` is set.
- GPL-3.0-or-later: "Applications that import or link this package and distribute the resulting binary should evaluate GPL obligations" (LICENSING.md); licensed features behind a key.
- Rapid surface growth (agent DAGs, harnesses, remote actions, incident/notification monitors) increases the reading load; legacy dir names (`children/`, `attempt_`) show format churn.

Scraping pipeline (fetch N pages per source, parse, enrich, publish snapshot): parent `refresh-sources` with `action: dag.run` + `parallel: ${params.SOURCES}` (`max_concurrent: 3`, `queue: scrape`); child `refresh-source` steps `fetch` (`run: fetch.sh ${params.host}`, `retry_policy: {limit: 3, interval_sec: 30, exit_code: [75]}`, `stdout: {artifact: pages/$ITEM.json}`), `parse` (`depends: fetch`, declared `outputs:` via `$DAGU_OUTPUT_FILE`), `enrich` (another `dag.run` `parallel` over event IDs or a `foreach` with `key`), `publish` (`preconditions` on coverage counts, `state.set` for the cursor). Plumbing you write: all fetch/parse/enrich programs and their idempotency; per-host throttling (a queue per host or sleeps); item keys and result assembly (`collect`); coverage metadata; artifact naming; cross-run reuse unless the pipeline can be expressed as regular files for `type: build`.

## Known limitations
- "`parallel:` currently requires `action: dag.run` to a child DAG." (llms.txt)
- "Build workflows are local-only. Distributed execution is rejected" (llms.txt, Build Workflows)
- "Human tasks cannot be used in sub-DAGs" (llms.txt)
- `max_active_runs` "DEPRECATED: This field is ignored for local (DAG-based) queues" (dag.schema.json)
- `max_output_size` default 1 MB, exceeding fails the step (dag.schema.json)
- Missed runs "are not replayed" without `catchup_window`; at most 1000 replayed (catchup.go, docs Scheduling)
- Zombie repair only for local runs; remote runs skipped (zombie_detector.go)
- DAG-level retries require a running scheduler, `retry_failure_window > 0`, latest attempt, top-level DAG (docs Durable Execution)
- Mixed Dagu versions must not execute the same run concurrently (README Paths)
- GPL licensing constraints for embedding (LICENSING.md)

## Relevance to Flowyard
- Borrow:
  - Attempt directories under a stable run ID, an append-only per-attempt status log compacted at the end, and a day index for listings — a clean file-layout precedent even if Flowyard uses SQLite.
  - Retry semantics: new attempt reuses the run ID; only failed/aborted nodes and their dependents reset; `--step`/`--downstream` targeted retry; "manual retry does not consume the auto-retry budget".
  - Scheduler watermark + `catchup_window` + `overlap_policy {skip, all, latest}` + `skip_if_successful` + deterministic scheduled-run IDs; `MaxMissedRuns` cap.
  - Liveness via heartbeat files with a stale threshold, and repair that first checks whether the PID identity is really dead.
  - Build-step reuse contract: staging path per attempt, atomic publish, refuse to overwrite externally changed outputs, `--no-reuse`, `dry` preview.
  - `exit_code` filters and `continue_on` as a small failure-classification vocabulary; `dagu schema` self-description.
- Avoid:
  - Shell-string data flow and env-scoped variables as the primary API; 1 MB inline output cap without a typed artifact reference.
  - Fan-out only by spawning child processes/DAGs; no in-process keyed map.
  - In-process retry waits that a crash converts into a failed attempt.
  - Feature accumulation across roles (agent DAGs, incident monitors, human tasks) in the same binary.
- Answers to open questions:
  - Q1: Retry rebinds recorded nodes to current step names and errors on a missing target step; there is no orphan record concept (runtime/plan.go).
  - Q2: Yes: `overlap_policy: skip` (default) for scheduled runs, per-DAG queue with `max_concurrency: 1`, `skip_if_successful`; keyed by DAG/queue name, not by parameter.
  - Q3: Both: `dagu start` is one-shot; `dagu scheduler`/`start-all` is the daemon that drains the file queue and spawns runs.
  - Q4: Steps run inline in the run's own process (or a worker's); DAG runs are dispatched through a persisted file queue drained by the scheduler (persis/store/queue.go, scheduler/queue_processor.go).
  - Q5: Only named queues with `max_concurrency` (persisted, cross-process); no per-host cooldown/rate limit.
  - Q6: One step = one process; no guidance beyond capping inline output at 1 MB and pushing big data to artifacts.
  - Q7: Inline in `status.jsonl` up to `max_output_size` (1 MB); logs and artifacts are files referenced by path.
  - Q8: No version gate; the attempt stores its DAG file and retry reloads the current file; build steps hash recipe+inputs+output.
  - Q9: `retryCount`/`retriedAt` per node and `autoRetryCount/Limit/Interval/Backoff` per run are persisted; next-retry time is derived by the scheduler scan, no jitter.
  - Q10: Off by default; with `catchup_window` a bounded chronological burst (<= 1000) subject to `overlap_policy`, `latest` yields "one current run".

## Sources
- https://github.com/dagu-org/dagu (clone, commit b1a5616a3d16, 2026-09-16) — README.md, llms.txt, ARCHITECTURE.md, LICENSING.md
- schemas/dag.schema.json (repo) — field descriptions for schedule, catchup_window, overlap_policy, skip_if_successful, queue, max_active_runs, max_active_steps, max_output_size, hist_retention_*, step retry_policy/repeat_policy/continue_on/preconditions/parallel/foreach
- internal/persis/file/dagrun/{dagrun.go,dataroot.go,store.go,attempt.go} (repo) — directory layout, status.jsonl, retention
- internal/persis/file/proc/store.go (repo) — proc heartbeat files
- internal/persis/store/queue.go (repo) — queue records
- internal/ir/{status.go,run_status.go,run_node.go} (repo) — statuses and persisted fields
- internal/runtime/{plan.go,node.go,stale_run.go} (repo) — retry plans, retry policy, stale-run repair
- internal/service/scheduler/{catchup.go,catchup_runid.go,scheduler.go,zombie_detector.go,retry_scanner.go,enqueue.go} (repo) — catch-up, zombie detection, DAG retries
- internal/schedulerstate/state.go (repo) — watermark state
- internal/cmd/retry.go (repo) — retry command
- api/v1/api.yaml (repo) — REST endpoints
- https://docs.dagu.sh/writing-workflows/durable-execution — step vs DAG retries (WebFetch summary with quotes)
- https://docs.dagu.sh/writing-workflows/scheduling — catch-up, overlap, one-off schedules (WebFetch summary with quotes)
- Not fetched: https://docs.dagu.sh/writing-workflows/sub-dags, /continue-on, /lifecycle-handlers, /incremental-workflows, /overview/web-ui (referenced by README; content taken from repo llms.txt instead).
