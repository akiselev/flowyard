# Research notes: synthesis

Written 2026-09-17 from fifteen notes in this directory. Fourteen were produced by four research agents working from primary sources (official docs, repository source, maintainer issues) on 2026-09-16; the Artifactum note was produced by a fifth agent reading `/projects/sinbad/artifactum` at commit `2c016f1`. Every note ends with a Sources section and lists its own unverified claims. Where this synthesis states a fact, the note it comes from is named in brackets; go there for the URL or the `path:line`.

Two maturity claims were re-checked directly against crates.io on 2026-09-17: `durare` is at 0.4.1 (created 2026-07-11, last release 2026-08-13) and `temporalio-sdk` is at 1.0.0 (released 2026-09-04).

## Index

| Note | Covers | Family | Lines | Sources |
|---|---|---|---|---|
| [temporal.md](temporal.md) | Temporal, Rust SDK 1.0 | history replay, server + workers | 271 | 38 |
| [restate.md](restate.md) | Restate, Rust SDK | journal replay, single-binary server | 212 | 23 |
| [obelisk.md](obelisk.md) | Obelisk | log replay, WASM sandbox, SQLite | 220 | 36 |
| [dbos.md](dbos.md) | DBOS Transact (Python, TS, Go) | named-step checkpointing, library, Postgres or SQLite | 182 | 29 |
| [absurd.md](absurd.md) | Absurd | named-step checkpointing, Postgres stored procedures | 177 | 31 |
| [inngest.md](inngest.md) | Inngest | named-step memoization over HTTP, server | 164 | 52 |
| [prefect.md](prefect.md) | Prefect 3.x | live execution plus opt-in result cache | 224 | 31 |
| [dagster.md](dagster.md) | Dagster | persisted execution plan, assets | 214 | 42 |
| [dagu.md](dagu.md) | Dagu | persisted YAML DAG, single binary, file state | 231 | 14 |
| [scrapy.md](scrapy.md) | Scrapy | persisted request frontier | 186 | 22 |
| [crawlee.md](crawlee.md) | Crawlee (JS and Python) | persisted request queue, autoscaled pool | 208 | 36 |
| [spider-rs.md](spider-rs.md) | spider crate | in-memory crawl loop | 164 | 18 |
| [rust-durable-crates.md](rust-durable-crates.md) | durare, apalis, underway, effectum, pgqrs | Rust journal and job-queue crates | 166 | 50 |
| [sqlite-durability.md](sqlite-durability.md) | SQLite WAL and synchronous, rusqlite vs sqlx | storage layer | 135 | 25 |
| [artifactum.md](artifactum.md) | Artifactum workspace, 58 crates | existing identity, CAS, action, attempt, pipeline planes | 283 | code |

## Execution-model families

The field sorts into four families. The design doc's provisional choice is the second.

| Family | Members | What it buys | What it costs |
|---|---|---|---|
| Replay of a positional history | Temporal, Restate, Obelisk, DBOS, durare | No step names; any control flow the SDK can replay; strong non-determinism detection (Temporal, Obelisk) | Every reorder is a breaking change with a patch ceremony; nondeterminism shows up as a stuck run; positional identity misattributes results after edits (DBOS, durare) |
| Named-step checkpointing | Absurd, Inngest, and the design doc | Reorders and insertions are safe; identity is readable in the database; loops work with explicit keys | Names must be invented and kept unique; both engines silently number repeats (`name#2`, `id:1`), which gives loops positional identity through the back door; a changed body under an old name is never recomputed |
| Persisted explicit structure | Dagster, Dagu, Artifactum pipeline | The plan is data, so inspection, retry-from-step, and partial re-execution are trivial; no replay rules at all | The structure must be known before execution; per-item fan-out needs a separate mechanism; not a function API |
| Persisted queue, whole-handler rerun | Crawlee, Scrapy, apalis, effectum | Simplest mental model; the request is the only unit; autoscaling and dedup are free | A crashed handler reruns with no record that it partially ran; results sinks duplicate rows; no cross-step state |

The three journal engines that enforce determinism hard (Temporal, Obelisk) do so with either a replay error that parks the run or a sandbox with no clock. The two library engines that do not (DBOS, durare) accept silent misattribution as the price of "no IDs". Named-step engines avoid both by making identity explicit and then undercut it with auto-numbering. That gap is exactly what the design doc's stricter key rule targets, and nothing surveyed occupies it yet.

## The ten open questions, answered by the field

Each row says what the surveyed systems do and what I would now decide for Flowyard. Question numbers match the review of DESIGN.md.

### Q1. Recorded operations the new code never reaches

- Temporal fails replay with a non-determinism error and the run stays stuck until patched or reset [temporal]. Obelisk raises `NondeterminismDetected("found unprocessed request stored at version ...")` and pins the run to the old component digest [obelisk]. Restate does not document it [restate].
- DBOS does not detect it; the positional shift causes `DBOSUnexpectedStepError` or silent misattribution [dbos]. durare silently orphans rows past the new code's end [rust-durable-crates].
- Absurd keeps them forever and ignores them, with a rename-on-change policy [absurd]. Inngest: "Memoized data persists but gets ignored", and a reorder is a warning [inngest].
- Dagster and Dagu snapshot the plan per run, so the case does not arise; a Dagu retry errors if a recorded step name is missing from the current file [dagster, dagu].

Decision: because Flowyard keys are explicit, unreached operations are detectable at run completion without replay errors. Record them as a warning on the run, show them in `inspect`, and fail only under a strict flag. Never reuse them silently and never number them.

### Q2. At most one active run per key

- Temporal: Workflow Id plus a conflict policy of Fail, Use Existing, or Terminate Existing [temporal]. Restate: a Virtual Object key is a single writer with a persisted per-key queue [restate]. DBOS: the workflow ID is the idempotency key and a second start "executes only once" [dbos]. Inngest: `singleton { key, mode: skip | cancel }` [inngest]. Dagu: `overlap_policy: skip | all | latest` plus per-DAG queues with `max_concurrency: 1` [dagu].
- underway enforces it in the database with a partial unique index on `(queue, concurrency_key)` where state is pending or in progress [rust-durable-crates]. Absurd returns the existing task for a repeated idempotency key [absurd]. Obelisk has nothing built in [obelisk]. Artifactum has no per-key lock anywhere [artifactum].

Decision: a workflow declares a singleton key derived from its input. Enforce it with a partial unique index on active runs, and expose a start policy of fail, attach, skip, or replace. Scheduled runs get the deterministic ID `sched/{name}/{instant}` so re-fires are idempotent, as in DBOS and Dagu.

### Q3. Daemon or one-shot process

- Daemons only: Temporal, Restate, Inngest, Obelisk, DBOS after `launch()`, Dagster [notes as named].
- Both modes from one code path: Dagu, where `dagu start` is one-shot and `dagu scheduler` drains a persisted file queue [dagu]; Crawlee, where `run()` returns and `keepAlive` turns it into a daemon [crawlee]; Prefect [prefect]. Absurd's Python SDK exposes a one-shot `work_batch()` [absurd].
- One-shot only: Scrapy, and every Artifactum CLI invocation [scrapy, artifactum].

Decision: one-shot `run` that takes the workspace lock, drains everything eligible, and exits. `serve` is a loop over the same function with a scheduler tick. Dagu shows this pairing works with a persisted queue; the design doc's rule that overdue work becomes eligible on start already assumes it.

### Q4. Activities inline in the orchestration future, or dispatched through a persisted queue

- Inline: DBOS, durare, Restate `ctx.run`, Absurd, Prefect tasks, Dagu steps, Scrapy, Artifactum's engine [notes as named].
- Dispatched: Temporal task queues, Inngest HTTP per step, Obelisk `-submit` rows pulled by executors, Crawlee's queue plus pool, Dagster's queue plus launcher plus subprocess [notes as named].

Decision: inline for activities, with workflow starts and map children in a persisted pending table. Restart recovery is then a sweep: everything marked running at startup is dead by definition under an exclusive owner, which is effectum's `JobRecoveryBehavior` and Crawlee's `single` ownership mode [rust-durable-crates, crawlee]. Artifactum needs this sweep too; nothing revisits its `running` attempt rows today [artifactum].

### Q5. Per-host cooldowns and limits

- Nobody persists a per-host cooldown. Inngest keys limits by an expression over the event and only sees the host if the event carries it [inngest]. DBOS computes queue rate limits from persisted start timestamps, which is the one cross-process design that needs no in-memory bucket [dbos]. Restate 1.7 persists concurrency limits keyed by scope and limit key [restate]. Dagster pools are static names with slots that leak on cancel [dagster]. Scrapy, Crawlee, and spider keep per-host state in memory and lose it on restart; Crawlee persists only its session pool [scrapy, crawlee, spider-rs].
- Artifactum has one global `--jobs` bound and lists per-origin limiting as future work [artifactum].

Decision: a `resource` table keyed by name, typically a host, holding `next_allowed_at` and a concurrency limit. A `Retry-After` response writes the host's `next_allowed_at`, not just the attempt's. Rate limits are computed from persisted attempt start times, DBOS-style. One short `BEGIN IMMEDIATE` write per update is cheap under WAL [sqlite-durability].

### Q6. What is cheap enough not to be a step

Every engine agrees: non-deterministic or side-effecting work goes in a step, pure computation stays in orchestration [dbos, restate, inngest, absurd, obelisk]. Temporal adds local activities for short idempotent work [temporal]. Prefect's scraper example splits fetch and parse "so both pieces can be retried or cached independently" [prefect].

Decision: same rule, plus a third category. Expensive pure work that should survive across runs is an Artifactum derive action, which gives cross-run cache hits for free [artifactum].

### Q7. Large outputs inline or by reference

- Inline with hard caps: Temporal 2 MB per payload and 50 MB per history; Inngest 4 MiB per step; Obelisk 1 MiB; Dagu 1 MB and the step fails [temporal, inngest, obelisk, dagu]. Inline without limits: DBOS, durare, Absurd [dbos, rust-durable-crates, absurd].
- Always by reference: Prefect result files, Dagster IO managers, and Artifactum, where outputs and even stdout are CAS ids [prefect, dagster, artifactum].
- Large transactions delay WAL checkpoints, so keeping the journal small has a storage cost as well [sqlite-durability].

Decision: the outcome record is a small inline value up to a threshold, and an artifact reference above it, chosen by the output type rather than by the author at each call site. This is the one place the design doc's "large outputs become artifact references" must be a rule.

### Q8. Gate on code change

- Explicit markers: Temporal `patched` and Build-ID pinning; DBOS `patch()` and durare `ctx.patch` [temporal, dbos, rust-durable-crates]. Deployment pinning: Restate immutable deployments; Obelisk content digest with replay-verified auto-upgrade and incompatible runs parked [restate, obelisk]. Source hash as version: DBOS default MD5 of workflow sources, fail-stop on mismatch [dbos]. Warn only: Inngest, Dagster `Unsynced` [inngest, dagster]. Nothing: Absurd, Dagu, Prefect, Crawlee [notes as named].
- Artifactum's action key includes declared code artifacts but nothing hashes the host binary [artifactum].

Decision: record the build digest per run. On continuation under a different digest, reject by default. Offer an upgrade command that does what Obelisk does automatically: re-enter the workflow and confirm every recorded operation is re-encountered with a matching fingerprint before allowing new work. Explicit keys make that check possible without a replay engine.

### Q9. Retry state that survives a crash

- Persisted attempt count and absolute next time: Temporal and Restate on the server; Absurd commits the failure and the next run row with `available_at` in one transaction; Obelisk in `t_state`; underway with a jitter factor; effectum; Dagu per node [notes as named].
- Not persisted: DBOS step retries, durare, Prefect (client-side), Scrapy (a counter inside the payload), Crawlee (count only, no time), spider [notes as named].

Decision: Absurd's shape. The failed attempt and the next attempt's absolute `available_at`, jitter included, commit together. This is the cheapest correct design in the survey and the one most often skipped.

### Q10. Missed scheduled runs

- Bounded catch-up with an overlap policy: Temporal catchup window plus Skip, BufferOne, BufferAll, and others; Dagu `catchup_window` plus `overlap_policy` plus a cap on missed runs; Dagster `max_catchup_runs=5` [temporal, dagu, dagster].
- Burst everything: DBOS `automatic_backfill`; Prefect, where every Late run executes on worker return [dbos, prefect]. One tick: Restate's cron pattern, Obelisk by inference [restate, obelisk].

Decision: a schedule declares `catchup: none | latest | window(d)` and `overlap: skip | queue | replace`. Defaults are latest and skip, which is the "one current refresh after a week asleep" behaviour the design doc asks for.

## Ergonomics: what the best APIs share

The user's stated priority is ease for recurring scraping and data-handling projects. These are the patterns that made the surveyed APIs pleasant, with the engine that does each best.

1. **An activity is an ordinary function.** DBOS, durare, Prefect, and the Temporal Rust SDK all decorate a plain async function and keep it directly callable in tests [dbos, rust-durable-crates, prefect, temporal]. Temporal's `#[activities]` on an impl block with `Arc<Self>` shared state is the best Rust shape seen.
2. **Capability lives in the context type.** Restate's `ObjectContext` versus `SharedObjectContext` makes exclusivity a type [restate]. Temporal's `&mut WorkflowContext<Self>` next to a separate `ActivityContext` is the same idea the design doc already proposes [temporal].
3. **No IDs on the happy path.** DBOS, durare, Restate `ctx.run`, and Crawlee need no step names [dbos, rust-durable-crates, restate, crawlee]. Every engine that achieves this pays with positional identity.
4. **Flow control is declared, not built.** Inngest's per-function `concurrency`, `throttle`, `debounce`, `singleton`, `priority`, and `batchEvents` with keys over the input is the benchmark [inngest]. DBOS's one `register_queue` call is the library version [dbos].
5. **Failure classification is an error type.** Inngest's `RetryAfterError` and `NonRetriableError`, Temporal's non-retryable error list, underway's retryable versus non-retryable errors [inngest, temporal, rust-durable-crates].
6. **Two failure hooks.** Crawlee's `errorHandler` before each retry and `failedRequestHandler` after exhaustion [crawlee].
7. **One context object per handler for scraping.** Crawlee hands each handler `request`, the parsed page, `enqueueLinks`, `pushData`, `log`, and `session`; a router with labels turns a crawl into a small state machine [crawlee].
8. **State you can `ls` or query.** Absurd's single SQL file, Restate's SQL introspection, Dagu's status files, Crawlee's storage directories [absurd, restate, dagu, crawlee].
9. **Operator verbs.** DBOS `list/get/steps/cancel/resume/fork`; Restate `pause/resume/restart-as-new/kill`; Dagu retry-from-step with `--downstream` [dbos, restate, dagu].

And the patterns that hurt most:

- Positional identity in any form, including auto-numbered repeats [dbos, durare, absurd, inngest].
- A mandatory server or HTTP hop per step [temporal, restate, inngest, dagster].
- Inline payload caps with no reference type [temporal, inngest, obelisk, dagu].
- Purge-by-default storage and append-only result sinks that duplicate on rerun [crawlee, scrapy].
- Retry allowance or cooldown kept in memory [dbos, prefect, scrapy, crawlee, spider-rs].
- Caching that silently does nothing until a second switch is flipped [prefect].

## Artifactum: what is already built

The full analysis is in [artifactum.md](artifactum.md). Its anchors were spot-checked against the code.

**Already there.** Content, artifact, and action identities with scheduling noise excluded from the action key. The pure, reproducible, volatile, and effect cache rules. An attempt row committed before execution and a realization committed after, with no transaction held across execution, which is the same commit ordering the design doc specifies. Named checkpoint artifacts re-materialized on retry. Cancellation by control file. Determinism audits, lineage, GC roots, effect receipts. A WAL SQLite file that is path-configurable and already shared by the evidence crate. A `foreach` planner where one item is one action identity.

**Missing.** Any run, operation, or schedule record. Any in-process Rust-function executor: all seven backends spawn a command in a materialized sandbox. Any retry policy, attempt counter per operation, or persisted next-retry time. Any workspace ownership lock. Any sweep of attempts left in the running state after a crash. Any per-key mutual exclusion or per-host limit. Any build identity for the host binary. Only one transaction exists in the whole metadata crate, and the connection is private, so nothing outside the crate can commit atomically with a realization. Refs are JSON files outside SQLite, so the publication pointer cannot ride the same transaction as the publish operation. An empty output collection is rejected, which a zero-item map would hit.

**The pipeline crate** is a Dagu-style level-parallel DAG executor. Its run object is never persisted; recovery after a crash is re-plan and hit the action cache. It is complementary to a Rust-function workflow API, not a competitor, and its `foreach` keyed-collection realization is the right shape for map outputs.

**Recommendation from the note, which I endorse.** Put Flowyard's run, operation, attempt, child, and schedule tables in the same SQLite file `artifactum-metadata` opens, add a small transaction escape hatch to that crate, extend `gc_roots` so Flowyard references keep artifacts alive, and drive `artifactum-engine` for activities. The three largest pieces of new code are the journal and re-entry engine, the in-process activity boundary with codecs and build identity, and the macro and tooling layer. Nothing in Artifactum covers the first; the second has storage hooks but no function-shaped execution; the third has per-attempt CLI verbs to reuse underneath.

So the honest answer to "how much is done": the artifact half, which is the half most projects get wrong. The workflow half does not exist, and a macro layer alone would not create it.

## The closest Rust precedent

durare is a DBOS-compatible durable execution crate with `#[workflow]` and `#[step]` on ordinary async functions, Postgres, SQLite, and in-memory providers, durable sleep, an idempotent start by workflow ID, and DB-backed queues with per-queue concurrency and rate limits [rust-durable-crates]. It proves the macro shape and the two-table journal are enough for a library. It is not a foundation to build on: identity is a per-execution counter guarded by name, so unreached rows are silently orphaned; steps take `&DurableContext`, so concurrent awaits on one context race for the same sequence number; step retries live in memory; recovery is opt-in and depends on executor-id conventions; the crate is nine weeks old with a single maintainer. Its macro code is worth reading for ergonomics, and its `UnexpectedStep` error wording and guarantee table are worth copying.

## API design questions to discuss

These are the decisions the research sharpened rather than settled. Each has a recommended default.

1. **Operation identity without strings.** Artifactum already derives action identity from content. Flowyard could do the same within a run: the default operation key is `(activity id, input fingerprint)`, and a string discriminator is required only when the same activity is called with the same input twice in one run, which the engine can detect at runtime and refuse with a clear error. That gives DBOS-style "no IDs" ergonomics with non-positional identity. The map primitive keeps `key_by`. Recommended.
2. **Where flow control is declared.** On the activity definition, as an attribute with a key expression over the typed input, in the Inngest style; at the call site; or in workspace configuration by resource name. Recommended: on the definition for the shape and default limits, with the resource name resolved from input, and workspace configuration able to override numbers.
3. **The failure type.** One `ActivityError` with `Transient { retry_after: Option<Duration> }`, `Terminal`, and `Uncertain`, where `Uncertain` parks the operation for reconciliation. The Retry-After variant also updates the host resource. Recommended.
4. **Fan-out shape.** A fixed-set `map` with recorded membership, Obelisk join-set style, covers feeds and calendars. A dynamic frontier where children add work, Crawlee style, is a second primitive that browser crawls will eventually want. Recommended: `map` only in the first version, with a bounded crawl as one activity that uses spider inside, as the design doc says.
5. **Output storage rule.** Type-driven: small serialisable results inline, anything marked as an artifact type by reference, with a size guard that fails loudly rather than silently switching. Recommended.
6. **The code-change gate.** Build digest per run, reject by default, verified upgrade as an explicit command. Recommended over any version attribute.
7. **Process model.** One-shot `run` and a `serve` loop over it. Recommended, and it removes the daemon and IPC surface entirely.
8. **Whether to reuse the Artifactum SQLite file.** Same file with a transaction escape hatch, per the note. The alternative is two durability domains and a permanent dangling-reference risk at GC.
9. **Scraping surface.** Whether Flowyard should offer a Crawlee-like handler context for acquisition activities, with page, enqueue, push, session, and log on one object, or leave that to the spider integration. Recommended: a small acquisition context in the activity layer, since the notes show that is where most scraping ergonomics come from.

## Discrepancies and open items

- The durare note describes the context as shareable; the docs.rs summary says it is not `Clone`. Both are consistent: it is borrowed by shared reference, which is what lets concurrent step awaits race. The design doc's `&mut` borrow avoids exactly this.
- Restate's default retry limit is reported as 70 by one page and unlimited by another; unverified [restate].
- Whether Inngest enqueues cron instants missed during server downtime is undocumented; the marketing claim covers app downtime only [inngest].
- No engine in the survey documents what happens to a step result that arrives after cancellation; Inngest states that executing steps run to completion. Flowyard's "attempt no longer authorized" check is therefore ahead of the field and should stay.
- Restate's Rust documentation pages returned 404 and the note relies on docs.rs and the SDK source [restate].
- The Artifactum e2e script has no dated record of passing at the current commit, and the engine, executor, pipeline, and CLI crates have no unit tests [artifactum].
