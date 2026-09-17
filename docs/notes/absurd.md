# Absurd

Researched: 2026-09-16. Scope: github.com/earendil-works/absurd main branch (latest release 0.5.0, published 2026-08-04), `sql/absurd.sql` (3150 lines), docs under `docs/` (concepts, comparison, storage, cleanup, living-with-code-changes, sdks/typescript, tools/absurdctl, tools/habitat), TypeScript SDK source `sdks/typescript/src/index.ts`, Python SDK README, issues #110, #111, #115, #126, #131, #136.

## Summary
Absurd (Armin Ronacher / earendil-works, Apache-2.0) is "the simplest durable execution workflow system you can think of. It's entirely based on Postgres and nothing else." The whole engine is one SQL file of plpgsql functions; SDKs (TypeScript, Python, experimental Go) are thin wrappers that pull tasks, run a handler, and write named checkpoints back. Tasks are retried as a whole with backoff; completed checkpoints are reused on the next attempt. It optimizes for a tiny surface area (tasks, steps, retries, sleep, events) and for being explainable, and it explicitly declines flow control, push delivery, event triggers and deterministic replay.

## Deployment and process model
- Library + schema. `absurdctl init` applies `absurd.sql`; `absurdctl create-queue <name>` creates per-queue tables. Workers are your own processes: `await app.startWorker()` (TS) or `app.start_worker(worker_id=, claim_timeout=, concurrency=, poll_interval=)` (Python) poll `absurd.claim_task`. Python also offers a one-shot `app.work_batch(worker_id, claim_timeout, batch_size=10)`.
- "Absurd is a pull-based system, which means that your code pulls tasks from Postgres as it has capacity. It does not support push at all" (README). No coordinator; Postgres is the only service.
- Something must be polling for a task to progress; sleeping tasks are not resident, they become rows with `available_at` in the future and are claimed again later.
- Single-node story: one worker process plus Postgres. No SQLite; the engine is plpgsql. Recommended scaling is "spawning multiple processes rather than relying on single-process concurrency settings" (Python SDK doc).
- TS `WorkerOptions { workerId, claimTimeout, batchSize, concurrency, pollInterval, onError, fatalOnLeaseTimeout }` ([index.ts L86-94](https://raw.githubusercontent.com/earendil-works/absurd/main/sdks/typescript/src/index.ts)).

## Execution model
Named-checkpoint reuse with whole-task rerun; no history replay and no determinism enforcement. Verbatim (docs/concepts.md): "once a step completes successfully, its return value is persisted in Postgres and the step will never execute again — even across process restarts or retries." "Code **outside** steps may execute multiple times across retries. Keep side-effects inside steps." "in Absurd, the system replays a task by **reusing checkpoints** and re-running ordinary code around them" (docs/comparison.md); "It does **not** try to turn your code into a deterministic workflow runtime."

Suspension is implemented by throwing: `sleepUntil` persists the wake time as a checkpoint, calls `schedule_run`, then `throw new SuspendTask()` ([index.ts L414-425](https://raw.githubusercontent.com/earendil-works/absurd/main/sdks/typescript/src/index.ts)); the handler unwinds, the run becomes `sleeping`, and a later claim reruns the handler from the top with checkpoints cached. Same for `awaitEvent`.

## Step and operation identity
- Identity = `(task_id, checkpoint_name)`, primary key of `c_<queue>`. Names are explicit strings. Repeated use of the same name in one run is auto-numbered by the SDK: `count === 1 ? name : `${name}#${count}`` (TS `getCheckpointName`, [index.ts L429-434](https://raw.githubusercontent.com/earendil-works/absurd/main/sdks/typescript/src/index.ts)); docs: "`handle.checkpointName` contains the concrete checkpoint key (`name`, `name#2`, ...) after Absurd's automatic step numbering." So loops without unique names silently get positional identity.
- Sleeps and waits are checkpoints too: `sleepFor(stepName, seconds)`, `awaitEvent(name, { stepName })`, `awaitTaskResult` under `$awaitTaskResult:<taskID>`.
- Mismatch: none detected. No input fingerprint, no type check; whatever JSON is under that name is returned. "A completed step returns its cached value forever." (docs/patterns/living-with-code-changes.md).
- Unreached recorded steps: they stay in `c_<queue>` until queue cleanup (`cleanup_ttl`, default 30 days after task completion per policy) and are simply not read. The code-change doc's advice is naming discipline: "If you are unsure whether a change is compatible, assume it is not and rename the step." Suggested conventions `fetch-user:v2`, `render-email#2026-04`.
- Stale-attempt writes: `set_task_checkpoint_state` only upserts when `v_new_attempt >= v_existing_attempt` (a newer attempt wins over an older worker's late write) and raises `AB002` if the writing run is already `failed` ([absurd.sql L1460-1545](https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql)).

### State machine (derived from absurd.sql)
Task `t_.state` and run `r_.state` share the enum `pending | running | sleeping | completed | failed | cancelled`.
- `spawn_task`: task `pending`, run 1 `pending`, `available_at = now`.
- `claim_task`: run `running` with `claimed_by`, `claim_expires_at`; task `running`, `attempts = max(attempts, run.attempt)`, `first_started_at` set once. Expired waits (`w_.timeout_at <= now`) are deleted in the same statement so the handler sees a timeout.
- `set_task_checkpoint_state`: upsert checkpoint, optionally extend the lease; refuses if the task is `cancelled` (AB001) or the run is `failed` (AB002).
- `schedule_run` (sleep): run `sleeping`, claim cleared, `available_at = wake time`. `await_event(queue, task_id, run_id, step_name, event_name, timeout)`: if the event already exists its payload is written as the step's checkpoint and returned; otherwise a `w_` row `(task_id, run_id, step_name, event_name, timeout_at)` is upserted and the run set `sleeping` with `available_at = timeout_at or 'infinity'` and `wake_event = event_name` (L1692-1875). A timed-out wait resumes with a null payload and is detected by `wake_event` matching with no payload.
- `complete_run`: run and task `completed`, `result`/`completed_payload` stored, waits deleted.
- `fail_run`: run `failed` with `failure_reason`; if `attempt + 1 <= max_attempts` a new run row is inserted at `available_at = now + retry_delay` (task `sleeping` if in the future, else `pending`); if the next attempt would exceed `cancellation.max_duration` the task is `cancelled`; otherwise the task is `failed`.
- `retry_task(queue, task_id, options)`: "Retries a failed task either by extending attempts on the same task or by spawning a brand new task from the original inputs" (`spawn_new`, `max_attempts` options; SQL comment L1327-1334).
- `cancel_task`: task and active runs `cancelled`, waits deleted.

Events: `emit_event` inserts into `e_<queue>` with `on conflict (event_name) do nothing` (first emit wins, L1879-1975) and wakes waiting runs. Payloads are immutable JSON. This is the subsystem the maintainer proposes removing (#136, below).

## Persistence schema
Source: [`sql/absurd.sql`](https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql), function `absurd.ensure_queue_tables` (L184-365). Per queue `<q>`:
- `t_<q>` tasks: `task_id uuid PK (UUIDv7), task_name, params jsonb, headers jsonb, retry_strategy jsonb, max_attempts int, cancellation jsonb, enqueue_at, first_started_at, state in (pending, running, sleeping, completed, failed, cancelled), attempts int, last_attempt_run uuid, completed_payload jsonb, cancelled_at, idempotency_key text unique` (unpartitioned) or a side table `i_<q>(idempotency_key PK, task_id)` (partitioned).
- `r_<q>` runs: `run_id PK, task_id, attempt int, state (same enum), claimed_by text, claim_expires_at, available_at, wake_event, event_payload jsonb, started_at, completed_at, failed_at, result jsonb, failure_reason jsonb, created_at`. Indexes on `(state, available_at)`, `(task_id)`, partial on `claim_expires_at where state='running'`.
- `c_<q>` checkpoints: `task_id, checkpoint_name, state jsonb, status text default 'committed', owner_run_id uuid, updated_at; PK(task_id, checkpoint_name)`.
- `e_<q>` events: `event_name PK, payload jsonb, emitted_at` (first emit wins, immutable). `w_<q>` waits: `task_id, run_id, step_name, event_name, timeout_at; PK(run_id, step_name)`.
- `absurd.queues(queue_name PK, storage_mode unpartitioned|partitioned, default_partition, partition_lookahead 28d, partition_lookback 1d, cleanup_ttl 30d, cleanup_limit 1000, detach_mode, detach_min_age 30d)`.
- Step outputs are inline `jsonb`; "The return value **must be JSON-serializable**" (TS SDK doc). No size limit or blob reference documented. Partitioned mode uses weekly range partitions by UUIDv7 for retention (docs/storage.md). Migrations ship in `sql/migrations` and via `absurdctl migrate --from X --to Y --dump-sql`.

## Crash recovery of in-flight work
- Claim = lease. `claim_task(queue, worker_id, claim_timeout=30, qty=1)` selects runs `state in (pending, sleeping) and available_at <= now ... for update skip locked`, sets `state='running', claimed_by, claim_expires_at = now + claim_timeout` ([absurd.sql L908-1071](https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql)). Every checkpoint write may extend the lease (`p_extend_claim_by`), and the SDK heartbeats via `absurd.extend_claim` on an interval (TS worker L567-595).
- Dead worker: the next `claim_task` call sweeps up to `qty` runs with `claim_expires_at <= now` and calls `fail_run` with reason `{"name": "$ClaimTimeout", "message": "worker did not finish task within claim interval", ...}`, which schedules a new attempt with backoff. Nobody declares a step idempotent or manual; the whole task is simply rerun and completed checkpoints are skipped.
- Documented overlap risk, verbatim (docs/concepts.md): "If a worker crashes or fails to make progress before the lease expires, the task becomes available for another worker to pick up. This means brief overlapping execution is possible — design your steps to tolerate it." And: "tasks should make observable progress well within the claim timeout and leave ample headroom."
- Late results: a stale worker's `complete_run` on a run already marked `failed` raises `AB002 "Run ... has already failed"`; its checkpoint writes are rejected the same way (L1100-1110, L1500-1505). `fatalOnLeaseTimeout` in the TS worker lets the process die instead.

Worker loop (TS `startWorker` doc comment, [index.ts L1102-1116](https://raw.githubusercontent.com/earendil-works/absurd/main/sdks/typescript/src/index.ts)), verbatim: "Tasks are claimed with a lease (claimTimeout) that prevents other workers from processing them. The lease is automatically extended when tasks write checkpoints or call heartbeat(). If a worker crashes or stops making progress, the lease expires and another worker can claim the task." Defaults: `concurrency` 1, `claimTimeout` 120 s, `batchSize` = concurrency, `pollInterval` 0.25 s, `workerId` `hostname:pid`. A background timer calls `SELECT absurd.extend_claim($1, $2, $3)` (L567-595); `fatalOnLeaseTimeout` makes a lost lease fatal to the process. Cancellation: "running tasks stop at the next checkpoint/heartbeat" (L1053).

## Retries, backoff, timeouts
- Retry unit is the task: "Retries happen at the **task** level, not the step level. When a task fails: 1. The current run is marked as failed 2. A new run is scheduled with backoff ... 3. The new run replays completed checkpoints and continues from where it left off" (docs/concepts.md).
- Policy is declared at spawn (or client/task defaults): `maxAttempts` (client default 5), `retryStrategy {kind: none|fixed|exponential, baseSeconds (fixed 60 / exponential 30), factor 2, maxSeconds}`. Delay = `least(base * factor^(attempt-1), max_seconds)`, all values capped at 86400 s; no jitter in SQL ([`retry_delay_seconds`, L56-130](https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql)). `fail_run(queue, run_id, reason, p_retry_at default null)` lets an SDK pass an explicit next time.
- Persisted: `t_.attempts`, `t_.max_attempts`, `t_.retry_strategy`, every run row with `attempt`, `failure_reason`, and the next run's `available_at` (absolute) written in the same transaction as the failure ([`fail_run`, L1181-1333](https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql)). The next-attempt row exists before the delay elapses, so a restart cannot reset the allowance.
- Failure classification: none built in; any thrown error fails the run. `max_attempts` exhausted leaves the task `failed`; `absurd.retry_task()` (0.2.0) reopens it in place or spawns a copy. Cancellation policies `max_duration` (since first start) and `max_delay` (never started) are evaluated inside `claim_task` and `fail_run`.
- Timeouts: no per-step timeout; `claim_timeout` is the only deadline, and lease expiry is reported as a failure of that attempt.

## Flow control
None. "Inngest has a number of built-in concepts that Absurd deliberately does not try to own: ... flow control primitives like concurrency limits, throttling, debouncing, rate limiting, prioritization, and batching" (docs/comparison.md). Worker-side `concurrency` only bounds in-process handlers; `batchSize` bounds a claim. Queues are namespaces with their own tables; ordering is `available_at, run_id`. A host cooldown would have to be expressed as `sleepFor` inside the task or a separate queue per host.

## Fan-out, child workflows, map
- Spawn from inside a task with `app.spawn(...)`, then `ctx.awaitTaskResult(taskID, { queue })`, "durably wait for another task's terminal result from inside a running task"; the child "must point to a **different queue** than the current task context queue" (TS SDK doc; changelog 0.3.0). Spawning is not itself checkpointed unless wrapped in a step, so a rerun without an idempotency key spawns again; the doc's answer is `idempotencyKey`.
- No map/join primitive, no recorded work set, no parent link column. Open issue #110 "Fan-out / Fan-in" (2026-04-19) asks for `when_all/when_any` and notes "Storing the child task IDs ... in the parent task seems wasteful, especially for large batches"; #126 asks whether 1000 small parallel subtasks can be awaited. No maintainer reply was visible via the issues API at fetch time.
- Sizes: unbounded `params`/`result` jsonb; cleanup by TTL.

## Versioning and code change policy
No version attribute, no digest, no warning. Policy is documented in docs/patterns/living-with-code-changes.md: old state is either "**left behind** with a new step name, or **translated forward** by compatibility code." Strategy 1 rename (`process-payment:v2`), Strategy 2 keep the name and normalize old shapes. "A task that sleeps forever and wakes up every day accumulates more compatibility risk than a task that does one job and finishes. When possible, prefer spawning fresh tasks for recurring work." Rolling deploys of new task names are covered in docs/patterns/deployments.md (not fetched); 0.4.0 changed workers to "defer claimed unknown tasks back into the queue instead of immediately failing runs" (CHANGELOG).

## Schedules, timers, missed runs
- Durable sleep: `sleepFor(stepName, seconds)` / `sleepUntil(stepName, date)` store the wake time as the checkpoint and call `schedule_run(queue, run_id, wake_at)` which sets the run `sleeping`, clears the claim and sets `available_at` ([L1135-1179](https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql)). 0.5.0 fixed SDK sleeps "to schedule wake-ups relative to the database clock".
- No cron/scheduler in the engine. The docs offer a recipe "Cron Jobs With Deduplication Keys" (docs/patterns/cron.md, not fetched) built on spawn-time idempotency keys; pg_cron is used only for maintenance jobs (`absurd.enable_cron`: partition provisioning, cleanup, detach planning). Missed-run policy is therefore whatever the external scheduler does; the dedup key makes a re-fired instant a no-op.

## Cancellation and late results
- `cancel_task(queue, task_id)`: task and non-terminal runs become `cancelled`, waits deleted ([L1976-2042](https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql)). "Running tasks detect cancellation at the next checkpoint write or heartbeat call and stop gracefully" (docs/concepts.md); `complete_run`, `set_task_checkpoint_state` and `extend_claim` raise `AB001 "Task has been cancelled"`. A step already executing runs to its end; its result is discarded by the error.
- No compensation hooks. `cancellation: { max_duration, max_delay }` at spawn time for automatic cancellation.

## Observability and tooling
- `absurdctl`: `init`, `schema-version`, `migrate` (`--dry-run`, `--to`, `--dump-sql`), `create-queue [--storage-mode partitioned]`, `queue-policy`, `drop-queue`, `list-queues`, `cron --enable/--disable`, `list-detach-candidates`, `detach-candidate`, plus `spawn-task`, `retry` and `install-skill` (README, docs/tools/absurdctl.md).
- Habitat: Go binary with embedded SolidJS UI "for monitoring Absurd tasks, runs, checkpoints, and events", connects straight to Postgres, port 7890 (docs/tools/habitat.md). 0.5.0 added a terminal-failure filter.
- Everything is plain SQL; the repo ships an agent skill so LLM tools can query state (README "Working With Agents"). No cache explanation (no cache concept).

## API ergonomics
Minimal example, verbatim from https://earendil-works.github.io/absurd/ (docs/index.md, TypeScript tab):
```typescript
import { Absurd } from 'absurd-sdk';

const app = new Absurd();

app.registerTask({ name: 'order-fulfillment' }, async (params, ctx) => {
  const payment = await ctx.step('process-payment', async () => {
    return { paymentId: `pay-${params.orderId}`, amount: params.amount };
  });

  const shipment = await ctx.awaitEvent(
    `shipment.packed:${params.orderId}`,
  );

  await ctx.step('send-notification', async () => {
    return {
      sentTo: params.email,
      trackingNumber: shipment.trackingNumber,
    };
  });

  return {
    orderId: params.orderId,
    payment,
    trackingNumber: shipment.trackingNumber,
  };
});

await app.startWorker();
```
Python shape: `Absurd(conn_or_url, queue_name="default", default_max_attempts=5)`, `@app.register_task("name")`, `ctx.step(name, fn)`, a `@step("name")` decorator alias of `ctx.run_step` for sync code, `ctx.sleep_for/sleep_until/await_event/await_task_result`, `begin_step/complete_step`, `app.spawn(name, params, max_attempts=, retry_strategy=, idempotency_key=, queue=)`, `app.start_worker(...)` (sdks/python/README.md).

Delightful:
- The whole mental model fits on one page: a task, named steps, rerun on failure. The README example is the real API.
- One SQL file; state is inspectable with `psql`; migrations are ordinary SQL you can dump.
- Spawn-time `idempotencyKey` returns the existing task (`created: false`) instead of erroring.
- `beginStep/completeStep` decomposed form for wrapping external loops (agent frameworks).

Painful / footguns:
- Step name reuse is silently numbered (`name#2`); a loop over pages with `ctx.step('fetch', ...)` gets positional identity.
- Retry is whole-task with one strategy; a 429 on one page cannot back off independently of a parse failure.
- No fan-out/join; parent must track child IDs itself and children must live on another queue (#110, TS doc).
- Events are the maintainer's own stated weak spot (below).
- Overlap after lease expiry is by design; the doc pushes it onto step authors.
- `ctx.step` bodies must return JSON; non-JSON values are lost.

Scraping pipeline sketch: `spawn('refresh-source', {sourceId}, {idempotencyKey: 'refresh:lib:2026-09-16'})`; inside, `ctx.step('fetch:page-1', ...)` per page with explicit names, `ctx.step('parse', ...)`, then either enrich items in-line as `ctx.step(`enrich:${event.key}`)` or spawn one child per item on an `enrich` queue with idempotency keys and `awaitTaskResult` each; `ctx.step('publish', ...)` writes the snapshot. The user writes: unique step names per page/item, host rate limiting (sleep or queue-per-host), child ID bookkeeping, JSON-only outputs, any snapshot storage.

## Known limitations
- Events: issue #136 "Remove Events?" by mitsuhiko, opened 2026-07-27, verbatim: "We don't use events and there are issues with them. The end result is that it's a feature with bugs and we don't really put time into it to fix it. Not sure if people care much about it? I think if I designed this better it might be more useful. Eg: a better designed event system might make awaiting results also work. Right now both things are in a sorry state." (https://github.com/earendil-works/absurd/issues/136). One user comment (2026-08-28) reports events work for waking a task but were unreliable as a cancellation signal.
- Deliberately not provided (docs/comparison.md): a separate orchestration service, deterministic replay, event/cron triggers as entry points, flow control (concurrency, throttling, debounce, rate limit, priority, batching), an integrated dev/dashboard platform. "It also means you get fewer built-in guarantees and fewer high-level primitives."
- Overlapping execution after lease expiry (docs/concepts.md).
- Fan-out/fan-in (#110), state/synchronization primitives (#111), immediate local execution to avoid poll latency (#131), migration blue/green questions (#115) are open, unanswered in the API output.
- Go SDK marked "(experimental bootstrap)" (README).
- Postgres only; no SQLite path.

## Relevance to Flowyard
- Borrow: the four-table shape (task, run/attempt, checkpoint, wait) with the next attempt row and its absolute `available_at` written in the same transaction as the failure; sleep stored as a checkpoint whose value is the wake time; `idempotency_key` at spawn returning the existing task; attempt-ordered checkpoint writes (`new_attempt >= existing_attempt`) and rejecting completion of a run already marked failed; treating "unknown task name" as defer-not-fail during rolling deploys; the honesty of the "brief overlapping execution is possible" sentence as a model for stating recovery guarantees; the naming-convention advice for checkpoint evolution.
- Avoid: silent `name#2` numbering for repeated step names; task-level-only retry policy for scraping; events as a generic subsystem nobody uses; leaving child bookkeeping to the application; JSON-only checkpoint values with no artifact reference.
- Q1 (unreached steps): kept forever, ignored, never validated; policy is rename-on-incompatible-change (living-with-code-changes.md).
- Q2 (singleton per key): spawn-time `idempotency_key` unique per queue returns the existing task; no "one active run per key after completion" rule beyond retention.
- Q3 (daemon vs one-shot): pull workers, typically long-running, but Python `work_batch` is an explicit one-shot claim-and-run.
- Q4 (inline vs queue): the task as a whole is dispatched via the persisted queue (`claim_task ... skip locked`); steps run inline in the handler.
- Q5 (rate limits): none; in-memory worker concurrency only.
- Q6 (granularity): "Keep side-effects inside steps"; everything else may rerun; no guidance on cheap computation beyond that.
- Q7 (large outputs): inline jsonb, must be JSON-serializable, no limit.
- Q8 (code-change gate): none; naming discipline.
- Q9 (retry state): attempt count and absolute next time persisted before the delay elapses; no jitter.
- Q10 (missed runs): out of scope; dedup key on the scheduled instant makes re-fires idempotent.

## Sources
- https://raw.githubusercontent.com/earendil-works/absurd/main/README.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/sql/absurd.sql (tables L184-365, spawn_task L757-906, claim_task L908-1071, complete_run/schedule_run L1073-1179, fail_run L1181-1333, set_task_checkpoint_state/extend_claim L1460-1608, cancel_task L1976-2042, retry_delay_seconds L56-130)
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/concepts.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/comparison.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/index.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/storage.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/patterns/living-with-code-changes.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/sdks/typescript.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/tools/absurdctl.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/docs/tools/habitat.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/sdks/python/README.md
- https://raw.githubusercontent.com/earendil-works/absurd/main/sdks/typescript/src/index.ts
- https://raw.githubusercontent.com/earendil-works/absurd/main/CHANGELOG.md
- https://earendil-works.github.io/absurd/concepts/ , /comparison/ , /patterns/living-with-code-changes/ , /sdks/python/ , /cleanup/ (rendered docs, fetched)
- https://api.github.com/repos/earendil-works/absurd/issues/136 (+ comments), /110, /111, /126, /131, /115
- https://api.github.com/repos/earendil-works/absurd/releases/latest (0.5.0)
- Not fetched: docs/patterns/cron.md, docs/patterns/deployments.md, docs/sdks/python.md raw (rendered version fetched instead), the announcement blog post (lead only)
