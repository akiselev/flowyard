**There is a defensible scope for our own core: one process owns a workspace, SQLite records execution, Rust functions define workflows, and explicitly keyed steps provide recovery.** We can have a substantially richer API and better tooling without bringing back distributed workers, actor generations, a workflow language, or a compiler.

The research points toward an important distinction:

> Simplify by removing execution models and deployment modes—not by omitting the bookkeeping that makes the remaining model correct.

I retrieved the **Research library hotspots** discussion. The initial goal was to discover community events and recurring groups around South Pasadena, collect their schedules, and assemble a useful social calendar. The earlier discussion proposed structured feed/calendar adapters, organizer and venue records, recurring-event handling, provenance, and duplicate reconciliation. That is a better first test than a generic “download three URLs” example.

I reviewed current primary documentation from Temporal, Restate, DBOS, Inngest, Obelisk, Prefect, Dagster, Dagu, and Scrapy, plus selected Absurd documentation, source, and maintainer discussions. This is comparative architectural research, not a claim that I audited or tested all these engines.

## 1. What existing engines teach us

### Temporal: take the execution contract seriously

Temporal’s central strength is an explicit separation between orchestration and activities. Workflow execution reconstructs state by replaying code against recorded history; operations must produce compatible commands. Network calls and other nondeterministic work belong in activities. Code changes that affect command order require a versioning strategy. 

The lesson is not that Flowyard needs Temporal’s entire execution model. It is that **“ordinary code that resumes after a crash” needs a precise definition**.

For Flowyard, we should clearly distinguish:

- Code that reconstructs orchestration and may run repeatedly.
- Activities whose completed outcomes are recorded.
- External actions that may have happened even when no outcome was recorded.

What I would not copy is support for arbitrary, long-lived orchestration across changing deployments, accompanied by a large replay and compatibility surface. Our first workflows can be finite refresh operations, with conservative rules for resuming old code.

Temporal’s complexity is not inherently a mistake. Much of it buys capabilities we can explicitly decline.

### Restate: ergonomic APIs do not eliminate concurrency constraints

Restate demonstrates useful developer-experience ideas: typed generated clients, service definitions, explicit contexts, and journaled operations. These are worth studying independently of whether we adopt its runtime. 

Its Rust SDK also exposes the limits unusually clearly. The `ContextSideEffects` documentation prohibits context operations inside a journaled `run` closure and warns that a `run` should be immediately awaited before other context operations to avoid nondeterministic interleaving during replay. 

That materially qualifies our earlier durare discussion: **the general problem is not unique to durare**. Allowing arbitrary Rust futures to share an orchestration context creates difficult replay semantics.

For Flowyard, I would borrow the explicit context boundary and generated metadata, but make the supported concurrency model narrower: sequential orchestration plus a few structured parallel operations. We should not pretend that any `tokio::spawn`, `select!`, or `FuturesUnordered` composition automatically becomes durable.

### DBOS and Absurd: a database-backed library is a credible architecture

DBOS is useful precedent for putting durable execution in an application library backed by a database rather than requiring a separate workflow-service deployment. Its recovery documentation distinguishes single-server restart recovery from recovering selected executors in a distributed deployment. 

That distinction is exactly where we can simplify: **Flowyard does not need to decide whether another machine is dead**. It needs to prevent two local owners from executing the same workspace and recover after the previous owner has actually exited.

Absurd offers another deliberately compact model: named checkpoints stored in PostgreSQL, tasks with successive execution attempts, and retries at the task level. A retry reruns the task while reusing completed checkpoints. Its documentation also acknowledges that lease-based recovery can briefly overlap executions. 

Two lessons follow.

First, a small core is plausible when its state transitions are concentrated in one transactional store. Second, retries at the whole-task level are simpler than a large hierarchy of independently retryable activities—but they can be too blunt for scraping, where different requests need different cooldowns and failure policies.

Absurd also provides a concrete warning against feature accumulation. In an open issue created on **July 27, 2026**, its maintainer reports that the event feature has bugs and receives little maintenance because they do not use it themselves. That is a maintainer-reported limitation, not a bug I reproduced. 

**Our lesson: no generic signal/event subsystem merely because workflow engines usually have one.** Add primitives because the first applications exercise them.

### Inngest: named checkpoints are attractive, but names are not semantic proof

Inngest identifies steps using explicit string identifiers and counters for repeated occurrences. Its versioning model can reuse completed step results across deployments, permit new steps, and warn rather than fail when step order changes. Changing a step’s body while retaining its identifier does not recompute an already completed result. 

This is an attractive alternative to treating a global sequence position as the primary identity.

However, permissive continuation is not necessarily what we want for evidence-processing pipelines. Suppose a parser call now receives a different page but accidentally retains an old invocation identity. “Keep running” is worse than a clear mismatch error.

For Flowyard, I would borrow **explicit named steps**, with stricter rules:

> Within an existing run, the same operation key must refer to the same operation type, input identity, and compatible output contract.

For collections, stable domain keys are preferable to occurrence counters. An event’s source identifier is a better identity than “the seventeenth event encountered.”

This is our proposed stricter policy, not a claim that Inngest’s documented behavior is a defect.

### Obelisk: structured parallel work deserves its own abstraction

Obelisk’s workflows use join sets to manage child executions. Submission persists an execution request, and joins provide an explicit relationship between a parent and its outstanding work. Its workflow model also makes replay and waiting part of the runtime contract. 

The valuable idea is **structured concurrency with durable membership**, not necessarily its WebAssembly component model.

For our common case:

```text
Given this fixed set of source events,
process each event independently,
then assemble the results in a defined order.
```

The engine should know that set, the child identities, and the completion policy. It should not infer them from whichever futures happen to finish first.

That gives us a useful, beautiful `map` API without supporting every possible asynchronous control-flow pattern.

### Dagster and Prefect: execution recovery and data reuse are different products

Dagster’s asset model centers persistent data and the code that produces it. That is closer to our question—“what event information is current and trustworthy?”—than a dashboard containing only successful and failed runs. 

Prefect’s caching documentation exposes several important details: caching requires persisted results; its default cache policy includes the flow-run identity; task-source hashing does not include nested task dependencies; and stronger isolation is needed to prevent concurrent duplicate computations under the same cache key. 

These are documented policies, not necessarily bugs. They illustrate why “we already store step results” does not mean “we already have a correct incremental computation cache.”

Flowyard needs both:

**Recovery:** do not repeat an already completed operation within this run.

**Reuse:** in a new run, avoid recomputing a parser or classifier when its inputs and relevant implementation are unchanged.

Artifactum should remain responsible for the latter. A second cache in the workflow core would recreate overlapping ownership.

### Dagu and Scrapy: simple operation is valuable, but “resume” needs qualification

Dagu demonstrates an appealing local operational package: a single binary, file-backed state, scheduling, workflow inspection, and explicit step definitions. Its documented durability model distinguishes step retries from new DAG attempts. It is useful precedent for deployment and tooling, although its declarative command-oriented interface is not the Rust function API we want. 

Scrapy supplies an especially relevant caution. Its persisted crawl jobs retain request queues, duplicate-filter state, and spider state, but the documentation says that unclean shutdown can corrupt a job directory; clean pause/resume is the supported path. 

Therefore, **crawler resumability is not automatically crash-safe workflow execution**. We must define our Spider recovery boundary ourselves rather than assume that retaining some crawler state solves it.

## 2. The execution model I would choose

There are three broad approaches worth distinguishing.

**Strict history replay:** rerun orchestration and verify that it emits the expected command sequence. Temporal is the clearest example.

**Named-step checkpointing:** rerun orchestration, but identify completed operations by durable keys and reuse their outcomes. Absurd and Inngest provide relevant variations.

**Explicit persisted workflow structure:** store steps and dependencies as data, then execute that structure. Dagu illustrates this family.  

For Flowyard, my provisional choice is:

> **Named-step checkpointing, conservative code compatibility, and structured child workflows.**

On restart, we rerun the Rust workflow function from its beginning. When it reaches a recorded operation, the engine returns the recorded outcome rather than executing the activity again. An incomplete operation is handled according to its persisted state and recovery policy.

We do **not** serialize Rust futures, preserve stack frames, transform arbitrary Rust into a persistent state machine, or build a custom deterministic executor.

This still imposes a rule: orchestration must depend on its recorded inputs and results. Reading the clock, a mutable configuration file, the network, or a randomly ordered collection can change what happens during re-entry. Those observations belong behind recorded boundaries.

### Use the type system to make the normal path safe

I would make `WorkflowContext` non-cloneable and have orchestration methods borrow it mutably across their calls:

```rust
async fn workflow(cx: &mut WorkflowContext, input: Input) -> Result<Output>
```

That makes casual parallel calls on the same context difficult to express.

An `ActivityContext` would be separate. It can provide HTTP clients, browser resources, Artifactum access, cancellation, and application dependencies—but not methods for opening nested workflow operations.

Parallelism goes through explicit helpers that create independent child scopes.

This does not prove arbitrary native Rust code is deterministic or side-effect-free. It makes the supported API enforce much more of the intended structure instead of relying entirely on documentation.

### We need persistent records, not a workflow language

We will need records for runs, operations, attempts, schedules, and parent–child relationships. Tooling can render an actual execution tree from them.

**That is not a reason to restore Flow IR.**

Those records describe execution that was requested or occurred. They need not form a general-purpose program representation, support multiple authoring languages, or be interpreted as a user-defined graph.

The useful part of the original design was discoverability, typed functions, evidence, and little application-specific plumbing. Those goals survive without KDL, a compiler, or the original service platform. 

## 3. Where a single-process core really becomes simpler

### Exclusive ownership replaces leases and failure detection

The deployment contract should be:

> One live Flowyard runtime owns one workspace, for its entire lifetime.

Enforce that with an operating-system-backed exclusive lock, not merely a PID file or a lease that expires.

A second process either communicates with the owner or refuses to start execution. It does not decide that the first process is “probably stalled” and take over. Multiple independent workspaces remain possible.

This removes distributed claiming, heartbeats, failover, lease renewal, leader election, and network-partition handling.

It does **not** eliminate late results after local cancellation. Each admitted attempt should still have an identity, and completion should be accepted only while that attempt remains authorized.

That is a local state check, not a distributed ownership protocol.

We also need honest cancellation semantics. Cancelling a future does not necessarily stop native blocking work or a spawned browser process. The owner must retain control of outstanding work and process lifetimes; it cannot release ownership while detached work still behaves as though it owns the workspace.

### SQLite should be the only workflow backend initially

I would use a serialized write path, short transactions, foreign keys, and WAL with `synchronous=FULL` by default.

SQLite permits concurrent readers but only one writer. WAL is designed around local shared-memory coordination, not arbitrary network filesystems. Its documentation distinguishes NORMAL’s application-crash protection from FULL’s stronger commit durability across operating-system crashes and power failures, subject to the storage system honoring synchronization. 

A single workflow owner and a single database writer are different properties. We need both; SQLite’s writer serialization does not stop two processes from making the same HTTP call.

I would not initially add PostgreSQL, a pluggable persistence trait, or an in-memory production backend. Test helpers can use temporary SQLite databases. One real backend is a substantial reduction in the correctness surface.

The database can contain current state plus an append-only attempt/audit history. We do not need to make the entire engine an event-sourced system whose every query reconstructs state from a log.

### The commit boundaries are the core

The essential sequence for an activity is:

```text
Persist the operation request and admitted attempt.
Commit.

Execute the activity without holding a database transaction.

Persist its outcome and the resulting state transition.
Commit.

Return the durable result to the workflow.
```

For child workflows:

```text
Persist the child and its parent association together.
Commit.
Only then make it executable.
```

This prevents the “child exists, but its parent has forgotten it” gap from becoming an untracked duplication mechanism.

For Artifactum outputs:

```text
Commit immutable output bytes and manifests.
Then record their references in the workflow database.
```

A crash between those stores can leave an unreferenced artifact. That is preferable to a successful workflow record pointing to content that was never committed.

Garbage collection must understand that recoverable workflows retain their referenced inputs and outputs. A workflow journal reference is not useful if an unrelated cache eviction can remove the referenced data.

### Identities must be explicit and separate

I would distinguish four identities:

| Identity | Meaning |
|---|---|
| Run ID | This particular request to refresh a source or build an agenda |
| Operation key | This logical call within that run |
| Artifactum action key | This computation on these inputs and implementation |
| External idempotency key | This particular effect against another system |

An operation key could be:

```text
refresh / source:library-calendar / enrich / event:abc123
```

It should not be derived from Tokio poll order, a source line number, or a single shared sequence counter.

When the same operation key is encountered during recovery, validate its operation type, input fingerprint, and contract. A mismatch should stop with a useful error. It should not silently return an old value, nor invent a different operation and execute it.

Repeated calls require distinct logical keys or child scopes. Fetching the same URL twice at different stages is not automatically the same observation.

The API can generate much of this structure. The application should supply stable domain keys where the engine cannot infer them safely.

## 4. What we still must implement correctly

Removing distribution is a major simplification. It does not remove these requirements.

### Interrupted activities need an explicit recovery policy

After a crash, an attempt can have no recorded outcome even though its external action occurred.

I would initially distinguish:

**Repeat-safe:** parsing, immutable artifact construction, and acquisition that the application explicitly permits repeating.

**Idempotent effect:** repeat using a stable key accepted by the destination.

**Uncertain effect:** stop for reconciliation rather than automatically repeat.

“Repeat-safe” does not mean “free.” Repeating a fetch still consumes requests and can trigger rate limits. Paid classification calls need their own cost policy.

For the first social-calendar application, we can avoid much of the hardest surface: acquire information and publish local agenda artifacts. Sending invitations, emails, or modifying an external calendar can come later.

### Retry state must survive a crash

Persist attempt counts before execution and absolute `next_retry_at` times after retryable failures.

A restart should not reset the retry allowance or erase a server-requested cooldown. Persist the selected retry time, including any jitter, rather than recalculating it on every restart.

The core needs a small distinction between transient failure, terminal failure, and an operation awaiting reconciliation. Application code supplies classification: an HTTP timeout, a 429 response, a changed page format, and a valid empty feed are different outcomes.

Source-wide cooldowns also differ from individual retries. Ten tasks targeting the same host should not each independently decide that their own delay has expired.

I would include basic named concurrency limits and cooldowns, but not build a general resource marketplace, quota language, or distributed rate limiter.

### Parallel mapping must record the work set

A durable `map` should record a fixed input collection or manifest, validate unique item keys, and create independently recoverable children.

Completion order should not determine the output order or the identities of later operations.

Initially, support a small number of explicit policies: collect all successful results, or collect typed per-item outcomes. Do not expose arbitrary races and promise that the engine will recover their meaning.

Also separate coordination from execution capacity. A parent waiting for its children must not occupy the only activity slot those children require.

### Timers and schedules should be ordinary persisted data

A durable timer records an absolute deadline. A schedule records when to create the next finite run.

For a single-process first version, waiting workflow futures can remain resident while the process runs; after restart, orchestration reconstructs itself from checkpoints. We do not need transparent workflow unloading and reactivation immediately.

Schedules need a declared missed-run policy. After a laptop has been asleep for a week, a community-events collector probably wants one current refresh—not a burst of every missed hourly refresh.

The schedule cursor and scheduled run creation must be committed together, with a deduplication identity for the scheduled instant. Nothing executes while the application is stopped; overdue work becomes eligible when it starts.

### Code changes should fail conservatively

Restate’s deployment guidance is a useful reminder that recorded executions remain coupled to compatible code; changing journal-producing behavior underneath an invocation can break recovery. 

For Flowyard’s first implementation, I would record the execution build identity and reject incompatible continuation.

The operator can finish with the old executable, or abandon that run and start a new one under the new code while explicitly reusing retained source artifacts.

That is less magical than hot migration, but much simpler and safer.

We should also avoid hashing only an annotated function’s body for cache validity. Its behavior can change through helpers, dependencies, configuration, or model versions. A conservative build digest is initially preferable to a clever but incomplete dependency fingerprint.

## 5. What the API could look like

This is a sketch to test the architectural direction, not a finalized or implemented API:

```rust
#[flowyard::workflow(version = 1)]
async fn refresh_source(
    cx: &mut WorkflowContext,
    source: SourceSpec,
) -> Result<SourceSnapshot> {
    let document = cx
        .call("acquire", AcquireSource, source.clone())
        .await?;

    let events = cx
        .call("extract", ExtractEvents, document)
        .await?;

    let enriched = cx
        .map("enrich", events, EnrichEvent)
        .key_by(|event| event.source_key.clone())
        .concurrency(4)
        .await?;

    cx.call(
        "publish",
        PublishSourceSnapshot,
        PublishInput {
            source_id: source.id,
            events: enriched,
        },
    )
    .await
}
```

The capitalized activity handles would be generated from ordinary function declarations. The original parser or normalizer remains directly callable in tests.

The intended ergonomics are:

**One function defines an activity.** Its declaration generates input/output metadata, the callable handle, and invocation adapters.

**A workflow reads like normal Rust.** Durable boundaries are visible without manual database writes or retry loops.

**Parallelism is one useful operation.** The author supplies an item key and concurrency limit, not a scheduler.

**Artifact handling is integrated.** Large outputs become typed artifact references without forcing every scraper author to hand-code storage plumbing.

I would retain separate semantics for derivations, observations, and effects, but avoid making them three unrelated frameworks. They can be policies on the same activity machinery.

### Tooling comes from the same definitions and records

The framework should expose:

```text
workflow describe refresh-source
workflow run refresh-source --source south-pasadena-library
workflow inspect <run-id>
workflow trace <run-id>
workflow retry <operation-id>
workflow explain-cache <operation-id>
```

The useful distinction is between **definition metadata**, **actual execution history**, and **artifact provenance**.

We can generate CLI arguments, schemas, JSON/JSONL output, help, and later a TUI from those sources. We do not need a static graph of every branch before execution.

“Retry” must mean creating an authorized new attempt, with an audit trail. It must not ambiguously mean replaying a stored failure and immediately returning the same error.

## 6. The library-hotspots use case should shape the boundaries

I would not implement it as one enormous workflow that crawls every venue and only publishes when everything succeeds.

Instead:

```text
Independently refresh each source
              ↓
Retain raw evidence and normalize source observations
              ↓
Publish a valid source snapshot with coverage metadata
              ↓
Build an agenda from a fixed set of source snapshots
```

That arrangement exercises the core without requiring persistent actors.

### Prefer source adapters over browser work

The prior discussion proposed calendar/feed/platform adapters before generic HTML or browser scraping. That remains the right first application structure.

Spider becomes an acquisition implementation, not the framework’s execution model.

For browser-heavy sources, the initial recovery unit should be a bounded crawl, page set, or explicit source cursor. We should promise that interrupted bounded work can restart and reuse retained evidence—not that we can restore a live Chrome session or arbitrary crawler frontier exactly.

### Completeness must have a scope

A calendar can be complete for “events visible between September and November” without representing the venue’s entire event history.

Therefore a source snapshot needs coverage and quality metadata, not merely `Vec<Event>`.

Absence can mean cancellation, a rolling date window, pagination failure, a parser regression, or an incomplete scrape. The application must not equate every missing event with deletion.

A failed or suspicious refresh should leave the previous valid snapshot available, with visible staleness and failure information. A workflow marked successful is not, by itself, proof that a source was exhaustively captured.

### Recurrence is a domain concern, not a generic deduplication rule

The iCalendar standard distinguishes recurring instances using identifiers including `UID` and `RECURRENCE-ID`; it also represents recurrence changes rather than simply listing unrelated timestamps. 

We should preserve those identities and exceptions, then normalize into event series and occurrences. Cross-source reconciliation should merge evidence without deleting the fact that multiple sources reported an event.

The workflow core should know none of that vocabulary. It should provide immutable inputs, keyed processing, recorded outcomes, and dependable publication.

### Publication needs one authoritative pointer

For this first application, a clean option is to store the current source-snapshot pointer in the workspace’s application tables and update it in the same SQLite transaction that accepts the publication operation.

Artifactum stores the immutable snapshot content. There should not be two competing “current” pointers in different stores.

If a later application keeps its authoritative state in another database, that boundary needs an idempotent publication protocol. We should not silently extend the local atomicity guarantee across stores.

## 7. The complexity budget I would adopt

**Keep:** typed workflows and activities, named operation identities, one SQLite backend, crash recovery, durable retries and deadlines, structured keyed children, cancellation state, Artifactum integration, source-level admission controls, and inspection.

**Exclude initially:** distributed execution, expiring ownership leases, multiple storage backends, persistent remote actors, arbitrary durable `select`, a generic event bus, automatic code migration, arbitrary workflow-language frontends, and serialization of live resources.

Most importantly, test the actual failure model before judging the API finished.

The acceptance suite should include real subprocess termination around attempt admission, external-call return, checkpoint commit, child creation, and publication. It should also test duplicate starts, different child completion orders, persisted retry limits, late completion after cancellation, corrupt stored outputs, incompatible code versions, and artifact retention across recovery.

For the first application, add source-specific tests: a truncated feed must not erase the agenda; an event leaving a rolling window must not automatically become cancelled; one broken source must not prevent other sources from becoming current.

A subprocess-kill test establishes more than an in-process parked future, but it still does not simulate power loss. Keep those claims distinct.

**My conclusion:** the strongest design is a **library-first, single-owner, checkpointed Rust workflow system**, with Artifactum providing cross-run data reuse and a small set of structured concurrency primitives.

That is more substantial than a macro wrapper, but much smaller than the original Flowyard platform. The additional complexity should go into **identity, transaction boundaries, recovery, backpressure, and diagnostic clarity**—the parts that make the easy API dependable—not into making the framework general enough to resemble every engine we researched.
