# Inngest

Researched: 2026-09-16. Scope: inngest.com/docs (TypeScript SDK v4-era pages using `triggers: {...}`; Python and Go mentioned where docs show them), the OSS server repo github.com/inngest/inngest main (`docs/SDK_SPEC.md`, `pkg/consts/consts.go`, `pkg/devserver/devserver.go`, `cmd/devserver/cmd.go`, `pkg/db/sqlite/{schema.sql,migrations.go,adapter.go}`, `pkg/execution/cron/manager.go`), LICENSE.md.

## Summary
Inngest is an event-driven durable-functions platform: your functions run inside your own HTTP app (Next.js, Node, Python, Go), and an Inngest server (cloud, self-hosted single binary, or the local Dev Server) calls them over HTTP, once per step, passing back memoized step results so the function re-executes from the top until it finds a step without a result. Steps are identified by explicit string IDs (hashed, with a counter for repeats). The server owns queues, flow control (concurrency, throttling, rate limiting, debounce, priority, batching, singleton), retries, cron, cancellation and the dashboard. It optimizes for product/backend teams who want queues and workflows without running workers, and for evolving long-running functions under permissive ("graceful") versioning. The server repo is SSPL-licensed with an Apache-2.0 grant three years after each release.

## Deployment and process model
- Two halves: (1) the SDK inside your web app, serving `/api/inngest`; (2) an Inngest server that stores events and run state and invokes the app. "Each step in your function is executed as a separate HTTP request" (https://www.inngest.com/docs/learn/how-functions-are-executed). Your app must be reachable; the server must be running.
- Dev Server: `npx inngest-cli@latest dev [-u http://localhost:3000/api/inngest]`, UI and API on :8288, auto-discovery of apps, an MCP endpoint at `/mcp` (https://www.inngest.com/docs/dev-server). Self-hosted: `inngest start` is "a single binary that includes all Inngest services" (Event API, event stream, runner, queue, executor, state store, GraphQL/REST APIs, dashboard UI) with SQLite at `./.inngest/main.db` and an in-memory Redis by default; `--postgres-uri`, `--redis-uri`, `--sqlite-dir`, `--queue-workers` (default 100) (https://www.inngest.com/docs/self-hosting).
- State split in the OSS binary ([`pkg/devserver/devserver.go`](https://raw.githubusercontent.com/inngest/inngest/main/pkg/devserver/devserver.go)): SQLite/Postgres via `pkg/db` for apps, functions, events, runs, history, traces; queue, run state, batches, pauses, concurrency/semaphores in Redis (`miniredis` in-process unless `--redis-uri`). `cmd/devserver/cmd.go` flags: `--persist` ("Persist data in between restarts"), `--sqlite-dir`, `--postgres-uri` ("Defaults to SQLite database"), `--no-discovery`, `--no-poll`. Without `--persist` SQLite is opened as `file:inngest?mode=memory&cache=shared` ([`pkg/db/sqlite/migrations.go` L49-69](https://raw.githubusercontent.com/inngest/inngest/main/pkg/db/sqlite/migrations.go)).
- OSS vs cloud: the OSS repo contains the executor, queue, flow control and cron manager, so the mechanics below are OSS. Cloud-only: the hosted dashboard features (Replay is described as a dashboard operation, https://www.inngest.com/docs/platform/replay), plan-based limits, support. The self-hosting doc names no feature list that is cloud-only; multi-node self-hosting was not available at 1.0 (announcement, lead only). Marked unverified beyond that.
- Single-node story: dev server or `inngest start --persist` on one box, app on another port. No library-only mode; there is always an HTTP hop per step.

## Execution model
Named-step memoization with re-execution from the top over HTTP; no history replay, no determinism verification, but a stack-order warning. Verbatim rule (how-functions-are-executed): "Any non-deterministic logic (such as DB calls or API calls) must be placed within a step.run() call to ensure it executes efficiently and correctly in the context of the execution model." Steps doc: "Place non-deterministic side effects, such as database writes or API calls, inside step.run() so Inngest can checkpoint the result and avoid re-running completed work during retries" (https://www.inngest.com/docs/learn/inngest-steps).

SDK spec ([SDK_SPEC.md](https://raw.githubusercontent.com/inngest/inngest/main/docs/SDK_SPEC.md)): "an SDK MUST maintain determinism in Call Requests when the underlying code has not changed, ensuring that two identical Call Requests will always produce the same output to the Inngest Server." Memoized data arrives in a `steps` object keyed by hashed step ID; "If a Step's hash is present in the `steps` object ... it MUST NOT be reported"; execution halts and reports when "a new Step is found that has no memoized result". Memoized errors are rethrown as `StepError`. Serialization caveat: "An SDK SHOULD make a Developer aware of this serialization process, as first-class entities such as class instances may be lost."

### Memoized values per op code (SDK_SPEC 5.3)
- `Step` (`step.run`): reported as `StepPlanned`, then executed in its own request when the server sends `?stepId=<hash>`; memoized as `{ data }` or `{ error }`. If the SDK cannot find the requested step "within a reasonable period" it returns `StepNotFound` (5.3.1).
- `Sleep`: "When the given time has elapsed, the Inngest Server will memoize the Step with `null`."
- `WaitForEvent`: "the Step will be memoized with the entire event payload ... If the timeout has elapsed without the Inngest Server receiving the event, the Step will be memoized with `null`."
- `InvokeFunction`: "memoize the Step with either a `{ data }` or an `{ error }` object depending on whether the invoked Function succeeded or failed."
- `sendEvent`: "An SDK MAY implement this as a wrapper around a Run Step ... Either approach is acceptable as long as the send is retriable and memoized."
- Parallelism (5.5): "When a Run reports >1 Step in response to a Call Request, `ctx.disable_immediate_execution` will be `true` for all subsequent Call Requests, meaning the SDK MUST then always report Steps with `StepPlanned` before executing them" (two round trips per step; `optimizeParallelism` reduces this).
- Large payloads (4.x): "requests with many batched Events or many memoized Steps may result in a payload that falls over the limit" so the SDK may fetch memoized step data with separate API requests (`use_api`).

## Step and operation identity
- Identity = SHA-1 of the human-readable ID; repeats get a counter, verbatim (SDK_SPEC 5.1.2): "each repeated human-readable identifier MUST append `:n` to the end of the identifier before hashing, where `n` is the number of times the Step has previously been found. For example, if `my-step-id` is repeated 3 times, the pre-hash identifiers for those steps should be `my-step-id`, `my-step-id:1`, and `my-step-id:2`." Docs: "the SDK also records a counter for each unique step ID. The counter increases every time the same step is called. This allows you to run the same step in a loop, without changing the ID."
- Mismatch / reorder (SDK_SPEC 5.4): the server sends `ctx.stack.stack`, the completion order; "If the SDK is following this ordering and the next Step cannot be found, it MUST first warn the user that the Function has appeared to change. ... Next, the SDK MUST continue to memoize Steps, but no longer expect them to sequentially match." A step the server asked to run that the code no longer reaches yields a `StepNotFound` op (5.3.1: "the Step could not be found due to the underlying code changing between reporting the Step and executing it, or a non-deterministic function inserting arbitrary delays").
- Unreached recorded steps: "Removing steps: Benign. Memoized data persists but gets ignored." Changing a step's body under the same ID does not re-run it; "To force re-execution of a step in in-progress runs, change the step ID" (https://www.inngest.com/docs/learn/versioning). No input fingerprint.
- Output limit: `MaxStepOutputSize = 4MB`, `MaxStepInputSize = 4MB`, `DefaultMaxStepLimit = 1_000` (absolute 10_000), `DefaultMaxStateSizeLimit = 32MB` ([`pkg/consts/consts.go`](https://raw.githubusercontent.com/inngest/inngest/main/pkg/consts/consts.go)); usage-limits page: "Step-returned payload size" 4MiB, 1000 steps per function (https://www.inngest.com/docs/usage-limits/inngest). Outputs are inline JSON in run state; no reference type.

## Persistence schema
- OSS Dev Server / self-host SQLite ([`pkg/db/sqlite/schema.sql`](https://raw.githubusercontent.com/inngest/inngest/main/pkg/db/sqlite/schema.sql)): tables `apps`, `functions`, `events`, `function_runs`, `function_finishes`, `history`, `event_batches`, `traces`, `trace_runs`, `spans` (+ indexes by run_id/status), `queue_snapshot_chunks`, `worker_connections`, `goose_db_version`. This is the CQRS/history side; live run state (memoized step outputs, stack) and the queue are in Redis via `pkg/execution/state/redis_state` (`queue_snapshot_chunks` suggests the dev server snapshots the queue into SQLite across restarts; mechanism unverified).
- Cloud: proprietary; not documented at table level.
- Retries and flow-control counters therefore live in the server's Redis, not with the SDK.

## Crash recovery of in-flight work
- The SDK is stateless; a crashed app process just fails the in-flight HTTP request for that step. The server retries the step request per the retry policy (`MaxFunctionTimeout = 2h` per request in consts). A step is re-executed unless its result was already reported and memoized. Nothing lets the user mark a step "uncertain"; you make it idempotent or catch the `StepError`.
- Server crash: state is in Redis/SQLite; in the dev server without `--persist` (and always for in-memory miniredis) it is lost. Self-hosted with external Redis/Postgres persists.
- No lease/heartbeat concept exposed; the request timeout is the deadline.

## Retries, backoff, timeouts
- Per step: "Each `step.run()` has its own independent retry counter"; default `retries: 4` (5 attempts; `DefaultRetryCount = 4`, `MaxRetries = 20` in consts), "exponential backoff with jitter" chosen by the server (https://www.inngest.com/docs/features/inngest-functions/error-retries/retries). The `attempt` argument is zero-indexed and resets per step.
- Classification: `throw new NonRetriableError(message, { cause })` stops retries; `throw new RetryAfterError(message, retryAfter: number|string|Date)` sets the next attempt time (e.g. from a 429 header) (https://www.inngest.com/docs/features/inngest-functions/error-retries/inngest-errors). After exhaustion the step throws `StepError` into the function, catchable to run a fallback step; uncaught, "the function itself will fail and terminate" and `onFailure` runs.
- Persistence: attempt counts and the next-attempt time are server-side queue state (Redis); the SDK sees only `attempt`. Whether jitter is persisted vs recomputed is not documented (unverified).
- Timeouts: function `timeouts.start` / `timeouts.finish` (create-function reference); `step.invoke` timeout default "1 year"; sleep up to one year.

## Flow control
All per-function configuration on `createFunction`, keys are CEL expressions over the triggering event (https://www.inngest.com/docs/guides/flow-control and sub-pages):
- `concurrency: { limit, key: "event.data.account_id", scope: "fn"|"env"|"account" }`, up to two constraints (`MaxConcurrencyLimits = 2`); "Concurrency limits the number of steps executing at a single time, not the number of function runs"; queued work is "best-effort FIFO". Sleeping/waiting runs do not count.
- `throttle: { limit, period (1s-7d), burst, key }`: delays starts, never drops. `rateLimit: { limit, period, key }`: GCRA; "Any events received in excess of your limit are ignored" (not queued). `debounce: { period, key, timeout }` (1 s to 7 d; runs with "the last event in the series"). `priority: { run: "event.data.account_type == 'enterprise' ? 120 : 0" }`, range -600..600 seconds of queue age. `batchEvents: { maxSize, timeout, key, if }`, 10 MiB hard cap, incompatible with idempotency, rate limiting, cancellation events and priority. `singleton: { key, mode: "skip"|"cancel" }`, incompatible with batching and explicit concurrency.
- Persisted in the server (Redis) and enforced across all app instances; scope `env`/`account` shares a key across functions. Nothing is per-host by default; a host key is `key: "event.data.host"` on a function that receives one event per request.

Function configuration surface (https://www.inngest.com/docs/reference/typescript/functions/create): `id` ("should not change between deploys"), `name`, `triggers` (`event`, `cron`, `if` CEL filter), `retries` (0-20, default 4), `concurrency`, `throttle`, `rateLimit`, `debounce`, `priority`, `idempotency` ("prevent duplicate events from triggering a function more than once in 24 hours"), `batchEvents`, `singleton`, `cancelOn`, `onFailure` ("called only when this Inngest function fails after all retries have been attempted"), `timeouts.start` / `timeouts.finish`.

## Fan-out, child workflows, map
- `step.sendEvent(id, event | event[])` (memoized, returns `{ ids }`) fans out to any functions triggered by those events; no result comes back (https://www.inngest.com/docs/reference/functions/step-send-event, fan-out guide).
- `step.invoke(id, { function, data, timeout })` calls one function and returns its result; "the invoked function will continue to run even if this step times out"; a failed invocation fails the step with `NonRetriableError` (https://www.inngest.com/docs/reference/functions/step-invoke).
- Parallel steps: create `step.run` promises without awaiting, then `Promise.all`; each step is its own request; with `optimizeParallelism` (v4 default) "Promise.race waits for all parallel steps to complete before resolving", and "racing steps are not cancelled" (https://www.inngest.com/docs/guides/step-parallelism). Python: `ctx.group.parallel()`.
- No map primitive; the work set is not recorded up front; identities come from your step IDs (`fetch-products-${pageNumber}` pattern, https://www.inngest.com/docs/guides/working-with-loops). Collection order is the array order you await; failure policy is per-step retry then `StepError`.

## Versioning and code change policy
Verbatim summary from https://www.inngest.com/docs/learn/versioning: "the SDK determines what to run based on the step identifiers in your code, not version numbers." "Completed steps are never re-executed, even across deployments." Adding steps: safe, they run when reached. Modifying step logic with the same ID: memoized result kept for in-progress runs. Changing an ID: forces re-execution. Removing steps: memoized data ignored. Reordering: "The SDK logs a warning when step execution order changes" and continues. For incompatible rewrites: route by event timestamp to a second function with an `if` trigger expression. There is no version attribute, no digest, no fail mode.

## Schedules, timers, missed runs
- `step.sleep(id, "2d")` / `step.sleepUntil(id, date)` are steps memoized with `null` when the time elapses; "up to a maximum of one year"; sleeping runs do not consume concurrency (https://www.inngest.com/docs/features/inngest-functions/steps-workflows/sleeps).
- Cron: `triggers: { cron: "TZ=Europe/Paris 0 12 * * 5" }`, optional `jitter: "5m"`; DST warning; free plan pauses a function after 20 consecutive failures (https://www.inngest.com/docs/guides/scheduled-functions). Overlap policy: none documented; combine with `singleton` or `concurrency: 1`.
- Missed runs: the product FAQ states "Inngest queues missed runs and executes them as soon as your deployment is available. Runs are never silently skipped." (https://www.inngest.com/uses/scheduled-jobs, marketing page, about the app being down). For the server itself: the OSS cron manager keeps one self-rescheduling job per (function, version, expression) in a system queue, with a periodic health check that "scans for functions that should have a pending cron job but do not" ([`pkg/execution/cron/manager.go`](https://raw.githubusercontent.com/inngest/inngest/main/pkg/execution/cron/manager.go), blog lead). Whether instants missed while the server was down are enqueued retroactively is not documented (unverified); the dev-server without persistence obviously loses them.

## Cancellation and late results
- `cancelOn: [{ event, if: "async.data.reminderId == event.data.reminderId", timeout }]` (up to five events), `timeouts.start/finish`, bulk REST `POST /cancellations`, dashboard. Verbatim: "Cancelling a function that has a currently executing step will not stop the step's execution. Any actively executing steps will run to completion." Cleanup via the system event `inngest/function.cancelled` (carries the error, original event, function id, run id) (https://www.inngest.com/docs/features/inngest-functions/cancellation).
- `singleton.mode: "cancel"` cancels the older run when a new keyed event arrives.

## Observability and tooling
- Dev Server UI: runs, step timeline, events, function list, replay of events, MCP endpoint for agents (dev-server doc). Self-hosted binary bundles the dashboard UI. Cloud dashboard: bulk Replay "from the Inngest dashboard" after fixing a bug (platform/replay), bulk cancellation, metrics, step metadata (`step-metadata` doc; OSS/cloud status unverified).
- REST/GraphQL APIs in the binary. No cache explanation concept.

## API ergonomics
Minimal complete example, verbatim from https://www.inngest.com/docs/guides/batching (TypeScript):
```ts
inngest.createFunction(
  {
    id: "record-api-calls",
    batchEvents: {
      maxSize: 100,
      timeout: "5s",
      key: "event.data.user_id",
      if: "event.data.account_type == \"free\"",
    },
    triggers: { event: "log/api.call" },
  },
  async ({ events, step }) => {
    const attrs = events.map((evt) => ({
      user_id: evt.data.user_id,
      endpoint: evt.data.endpoint,
    }));
    return await step.run("record-data-to-db", () => db.bulkWrite(attrs));
  }
);
```
Parallel steps, verbatim (step-parallelism guide): `const sendEmail = step.run("confirmation-email", async () => { ... }); const updateUser = step.run("update-user", async () => { ... }); const [emailID, updates] = await Promise.all([sendEmail, updateUser]);`. Official SDKs: TypeScript, Python, Go (https://www.inngest.com/docs/sdk/overview); no Rust SDK.

Delightful:
- Flow control is declarative config next to the function: concurrency per key, throttle, debounce, singleton, priority, batching, idempotency, cancelOn, with CEL keys over the event; nothing to build.
- `RetryAfterError` maps a 429 straight onto the next attempt time; `NonRetriableError` is one line.
- Step IDs plus counters let loops work; adding a step mid-run is safe; a warning rather than a crash on reorder.
- Dev Server is one command with a UI and event replay; MCP for agent-driven debugging.

Painful / footguns:
- Every step is an HTTP round trip through the server; function code outside steps re-runs on each request (execution model). Local-only, library-only use is not an option.
- Explicit string IDs everywhere; reuse in a loop gives positional `:n` identity, so a changed page order silently maps old results to new pages (versioning doc: "Use descriptive, stable, unique step IDs").
- Same ID + changed body never recomputes; stale results survive deploys (versioning doc).
- 4 MiB per step output, 1000 steps per run, JSON-only results (`Date`/`ObjectId` become strings) (usage limits, step-run reference, SDK spec).
- Idempotency keys expire after 24 h (handling-idempotency); `rateLimit` drops events rather than queuing (rate-limiting guide).
- Dev server state is in-memory unless `--persist`; queue/state lives in Redis even then (source).
- SSPL license for the server (https://github.com/inngest/inngest/blob/main/LICENSE.md).

Scraping pipeline sketch: `refresh-source` function triggered by `source/refresh.requested` with `singleton: { key: "event.data.source_id", mode: "skip" }` and `concurrency: [{ key: "event.data.host", limit: 2 }]`; `step.run(`fetch:${pageId}`)` per page (page list first recorded by a step); `step.sendEvent` one `page/parsed` event per item plus an `enrich-item` function with `throttle: { key: "event.data.host", limit: 10, period: "1m" }`; a final `publish` function invoked with `step.invoke` after `Promise.all` of `step.invoke("enrich:"+key, ...)` per item if results are needed in-line. The user writes: unique step IDs per page/item, the event schemas, the split into functions for per-host throttling, snapshot storage (results are inline JSON), the HTTP endpoint and the server process.

## Known limitations
- 4 MiB step output / 4 MiB step input / 1000 steps default (consts; usage-limits).
- Reorder and removal are warnings, not errors; same-ID body changes are not recomputed (versioning doc).
- Cancellation does not interrupt a running step (cancellation doc).
- `rateLimit` skips events; `singleton` incompatible with batching and explicit concurrency; batching incompatible with idempotency, rate limit, cancellation events, priority (guides).
- Missed cron instants during server downtime: undocumented (marketing claim covers app downtime only).
- No official Rust SDK; server is SSPL.

## Relevance to Flowyard
- Borrow: explicit step IDs with a documented `id:n` counter rule (but prefer domain keys); the `RetryAfterError` / `NonRetriableError` pair as the failure-classification API; keyed flow control expressed as declarative config (`key`, `limit`, `period`, `burst`, `mode`); the cancellation statement "actively executing steps will run to completion" as honest semantics; `singleton { key, mode: skip|cancel }` as the shape of a per-key rule; sleep as a memoized step with `null`; `StepError` catchable in the orchestrator for fallbacks; a system `function.cancelled` event for cleanup.
- Avoid: warn-and-continue on step-order change and silent reuse under a changed body; positional counters for loop identity; HTTP-per-step execution; in-memory queue/state for a local tool; 24 h idempotency windows; dropping events on rate limit.
- Q1 (unreached steps): "Memoized data persists but gets ignored"; reorder produces a warning only (versioning doc; SDK_SPEC 5.4).
- Q2 (singleton per key): yes, `singleton: { key, mode }` per function (skip or cancel); plus `idempotency` CEL key (24 h) and event `id` dedup (24 h).
- Q3 (daemon vs one-shot): always a server daemon plus an HTTP-served app; no one-shot mode.
- Q4 (inline vs queue): every step is dispatched by the server's persisted queue as a separate HTTP call; nothing runs inline in a resident orchestration future.
- Q5 (rate limits): server-side (Redis), cross-process, keyed by CEL expressions on event data, scoped fn/env/account; not keyed by host unless the event carries it.
- Q6 (granularity): non-deterministic side effects go in steps; pure code outside re-runs each request, so anything expensive should be a step, anything cheap should not (execution model).
- Q7 (large outputs): inline JSON, 4 MiB per step, 32 MB state per run (consts).
- Q8 (code-change gate): none; graceful continuation with warnings; timestamp-routed second function for rewrites.
- Q9 (retry state): server-side per-step attempt counter with exponential backoff and jitter; `RetryAfterError` sets an absolute time; persistence details are the server's (unverified beyond "Redis").
- Q10 (missed runs): app downtime: queued, "never silently skipped" (marketing FAQ); server downtime: unverified; overlap prevention via singleton/concurrency.

## Sources
- https://www.inngest.com/docs/learn/how-functions-are-executed (and /docs-markdown/ variant) (execution model)
- https://www.inngest.com/docs/learn/inngest-steps (step IDs, counter, memoization)
- https://www.inngest.com/docs/learn/versioning (step versioning, warnings)
- https://www.inngest.com/docs/guides/working-with-loops (loop IDs)
- https://www.inngest.com/docs/guides/flow-control , /concurrency , /throttling , /rate-limiting , /debounce , /priority , /batching , /singleton , /handling-idempotency
- https://www.inngest.com/docs/features/inngest-functions/error-retries/retries and /inngest-errors
- https://www.inngest.com/docs/usage-limits/inngest
- https://www.inngest.com/docs/guides/invoking-functions-directly , https://www.inngest.com/docs/reference/functions/step-invoke , /step-send-event , /step-run
- https://www.inngest.com/docs/guides/step-parallelism , https://www.inngest.com/docs/guides/fan-out-jobs
- https://www.inngest.com/docs/guides/scheduled-functions , https://www.inngest.com/docs/features/inngest-functions/steps-workflows/sleeps
- https://www.inngest.com/docs/features/inngest-functions/cancellation , /cancellation/cancel-on-events , https://www.inngest.com/docs/reference/system-events/inngest-function-cancelled , https://www.inngest.com/docs/guides/cancel-running-functions
- https://www.inngest.com/docs/reference/typescript/functions/create (config options)
- https://www.inngest.com/docs/dev-server , https://www.inngest.com/docs/self-hosting , https://www.inngest.com/docs/platform/replay , https://www.inngest.com/docs/sdk/overview , https://www.inngest.com/docs/features/inngest-functions/steps-workflows/step-metadata-how-to
- https://raw.githubusercontent.com/inngest/inngest/main/docs/SDK_SPEC.md (5.1.2 hashing/counter, 5.2 memoization, 5.3.1 StepNotFound, 5.4 stack warning)
- https://raw.githubusercontent.com/inngest/inngest/main/pkg/consts/consts.go (limits)
- https://raw.githubusercontent.com/inngest/inngest/main/pkg/devserver/devserver.go , /cmd/devserver/cmd.go , /pkg/db/sqlite/schema.sql , /pkg/db/sqlite/migrations.go , /pkg/db/sqlite/adapter.go , /pkg/execution/cron/manager.go
- https://github.com/inngest/inngest/blob/main/LICENSE.md
- Leads only: https://www.inngest.com/uses/scheduled-jobs (marketing FAQ on missed runs), https://www.inngest.com/blog/how-we-used-inngest-queues-to-build-inngest-native-cron-scheduler, self-hosting announcement blog (search snippet)
- Tried and failed (404): https://www.inngest.com/docs/guides/step-versioning (content lives at /docs/learn/versioning)
