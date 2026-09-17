# Artifactum (existing crate workspace)

Researched: 2026-09-16. Repo: `/projects/sinbad/artifactum`, commit `2c016f1` (2026-09-04).
All `path:line` references below are relative to that repo root. Nothing was built or run; every
claim comes from reading the files listed under Sources. Inferences are marked "(inference)".

LOC of the crates relevant here (Rust only, `find ... -name '*.rs' | xargs wc -l`):

| crate | LOC | crate | LOC |
|---|---|---|---|
| artifactum-core | 659 | artifactum-store | 1163 |
| artifactum-metadata | 309 | artifactum-receipt | 379 |
| artifactum-action | 161 | artifactum-evidence | 2891 (1628 lib + 1263 tests) |
| artifactum-engine | 747 | artifactum-resolver | 972 |
| artifactum-executor | 635 | artifactum-cli | 1146 |
| artifactum-pipeline | 853 | artifactum-provider-sdk / -api | 335 / 60 |

Every non-provider crate is a single `src/lib.rs`; the 50 `artifactum-provider-*` crates are
mostly 14-30 line shims over OpenDAL or the SDK.

## Summary

Artifactum is a content-addressed artifact lifecycle system: a SHA-256 CAS on disk
(`crates/artifactum-store`), a SQLite "history plane" of actions, attempts, realizations,
source observations, attestations and checkpoints (`crates/artifactum-metadata`), an engine that
turns an `ActionSpec` into a sandboxed subprocess attempt and imports its declared outputs
(`crates/artifactum-engine`), seven process-spawning executor backends
(`crates/artifactum-executor`), a TOML DAG runner with per-item `foreach` (`crates/artifactum-pipeline`),
a provider/plugin system for fetching external sources, and receipt/evidence crates that record
externally executed activities as intrinsic realizations with sealed, re-verifiable claims. The
workspace was LLM-generated from an earlier resolver-only codebase (`GENERATION_NOTES.md:3-7`),
later made to build (`git log`: `64bfccd`, `77b296d`), and its correctness argument rests on one
e2e shell script plus 38 unit/integration tests, 17 of which belong to the evidence crate
(`STATUS.md:96`). Engine, executor, pipeline, action and CLI have zero unit tests.

How much of Flowyard is already done: **the artifact half, not the workflow half.** Artifactum
already has the identities Flowyard wants to delegate (ContentId/ArtifactId/ActionKey with
scheduling noise excluded), the pure/reproducible/volatile/effect cache rules, an attempt record
committed before execution and a realization committed after, immutable stdout/stderr and effect
receipts, named checkpoint artifacts, lineage, GC roots, and a SQLite file that could host more
tables. It has **no** run or operation records, no in-process Rust-function executor, no retry
policy or persisted retry schedule, no workspace ownership lock, no restart sweep of orphaned
"running" attempts, no scheduler, no code/build gate, and no per-key mutual exclusion. The
pipeline crate is a level-parallel declarative DAG executor whose only recovery mechanism is
"re-plan and hit the action cache"; it is complementary to a Rust-function workflow API, not a
substitute. A Flowyard layer therefore has to build the entire run/operation/attempt journal,
the recovery and retry state machine, and the typed activity boundary; it can reuse Artifactum
for content identity, cross-run reuse, output storage, and provenance.

## Crate map

| crate | LOC | responsibility | maturity signal | Flowyard relevance |
|---|---|---|---|---|
| artifactum-core | 659 | I/O-free identity types: `ContentId`, `ArtifactId`, `ActionKey`, `ActionSpec`, `AttemptRecord`, `Realization`, `Checkpoint`, `CachePolicy` | 4 unit tests (`core:607-659`); everything depends on it | high |
| artifactum-metadata | 309 | SQLite plane; one `Mutex<Connection>`; 8 tables | 1 unit test (`metadata:279-308`); exercised by e2e via every CLI call | high |
| artifactum-action | 161 | `ActionBuilder`, structural `diff` for `why` | no tests; used by engine `why` (`engine:479`) | medium |
| artifactum-engine | 747 | cache lookup, sandbox, attempt, realization, checkpoints, cancel, retry, lineage | no unit tests; e2e steps 1-8 (`scripts/e2e_observe.sh:87-201`) | high |
| artifactum-executor | 635 | `Executor` trait + local/bwrap/container/ssh/slurm/k8s/plugin backends; all spawn processes | no tests; only `local` is exercised by e2e | high (the missing in-process executor lives here) |
| artifactum-pipeline | 853 | `Artifactum.toml` v3 model, lockfile, level planner, `foreach`, level-parallel runner | no unit tests; e2e steps 1-4 | medium (design contrast) |
| artifactum-store | 1163 | CAS, trees, collections, chunked blobs, refs, leases, GC, materialization | 7 tokio tests (`store:1020-1125`) incl. lease-vs-GC and corruption | high (output storage, GC roots) |
| artifactum-receipt | 379 | `ReceiptEnvelope<P>` with producer/activity identity and canonical receipt id | 5 unit tests (`receipt:259-379`) | medium (code-identity vocabulary) |
| artifactum-evidence | 2891 | assets with declared digests, run collections, sealed claims, `verify_claim`, `explain` | 17 tests in 3 files; the only crate with a written validation record (`STATUS.md:43-46`) | medium (pattern for "record an activity that ran outside the executor") |
| artifactum-resolver | 972 | `ArtifactProvider` trait, resolution, acquisition, source observations | tested indirectly by e2e fixture provider | low |
| artifactum-cli | 1146 | clap CLI: plan/run/exec/status/runs/lineage/why/audit/checkpoint/... | e2e drives it | medium (inspection surface to mirror) |
| artifactum-provider-api/-sdk | 60 / 335 | HTTP JSON helper; `serve_provider` plugin adapter | 1 fixture test | low |

## Identity model as implemented

- `ContentId` is SHA-256 over exact bytes (`crates/artifactum-core/src/lib.rs:70-83`, hashing at `core:594-605`).
- `ArtifactId` is SHA-256 over the canonical JSON of `ArtifactManifest { version, content, kind, media_type, schema, format_version, annotations }` (`core:85-98`, `core:305-332`). Provenance is deliberately absent from the manifest.
- `ActionKey` is SHA-256 over an `Identity` projection of `ActionSpec` (`core:495-525`). **Included:** `version`, `command` argv, `inputs` (name to ArtifactId), `code` (name to ArtifactId), `parameters` (arbitrary JSON), `environment` (variables + container ref), `outputs` (name to `OutputSpec { kind, media_type, schema }`), `network`, `sandbox`, `platform`. **Excluded:** `name`, `resources`, `budget`, `cache`. The exclusion is tested at `core:625-637`. The full `ActionSpec` is at `core:447-474`.
- Code identity is **only** `spec.code`: artifacts the caller chose to hash (`core:455`). The pipeline fills it by importing the files listed under a task's `code = [...]` from the project directory (`crates/artifactum-pipeline/src/lib.rs:760-769`). Nothing hashes the running executable, the crate graph, or a build. The receipt crate's `ProducerIdentity { repository, commit, package, package_version, executable: ArtifactId }` (`crates/artifactum-receipt/src/lib.rs:54-60`) is the closest existing vocabulary for a build identity, and the evidence crate puts `producer.executable` into `spec.code["executable"]` (`crates/artifactum-evidence/src/lib.rs:759-762`).
- Attempt id is a fresh `Uuid::new_v4()` per execution (`crates/artifactum-engine/src/lib.rs:303`); `AttemptRecord { id, action, executor, started_at, finished_at, exit_code, stdout: Option<ContentId>, stderr, metrics, error }` (`core:539-557`).
- `Realization { id, action, attempt, created_at, outputs: BTreeMap<String, ArtifactId> }` (`core:559-566`) exists only after success (`engine:409-416`).
- `CachePolicy { Pure, Reproducible (default), Volatile, Effect }` (`core:353-361`). Engine rules, all in `run_inner`:
  - cache hit only when `!force && matches!(cache, Pure | Reproducible)` and the latest realization's outputs are still loadable from the CAS (`engine:181-194`, `engine:431-438`);
  - `Volatile` and `Effect` always execute (they fall through the same `if`);
  - `Effect` with no declared outputs gets a synthesized `receipt` output artifact of media type `application/vnd.artifactum.effect-receipt+json` containing action key, attempt id, exit code and stdout/stderr content ids (`engine:395-408`);
  - `Pure` additionally asserts that every realization in history produced identical output ids, erroring with `Nondeterministic` otherwise (`engine:421-423`, `engine:490-501`).
- Per-name history: the engine writes `kv["last-action:<name>"]` and `kv["previous-action:<key>"]` so `why` can diff the previous spec of the same task name (`engine:170-180`, `engine:468-489`). Task names are therefore an out-of-key lookup index, not identity.

## Metadata plane schema

Source: `crates/artifactum-metadata/src/lib.rs:64-84`, executed on every `open` via `execute_batch`.

```sql
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;
CREATE TABLE actions(action_key TEXT PRIMARY KEY, spec_json TEXT NOT NULL, created_at TEXT NOT NULL);
CREATE TABLE attempts(id TEXT PRIMARY KEY, action_key TEXT NOT NULL, record_json TEXT NOT NULL,
                      started_at TEXT NOT NULL, finished_at TEXT, status TEXT NOT NULL);
  INDEX idx_attempts_action(action_key, started_at DESC)
CREATE TABLE realizations(id TEXT PRIMARY KEY, action_key TEXT NOT NULL, attempt_id TEXT NOT NULL,
                          record_json TEXT NOT NULL, created_at TEXT NOT NULL);
  INDEX idx_realizations_action(action_key, created_at DESC)
CREATE TABLE realization_outputs(realization_id TEXT NOT NULL, name TEXT NOT NULL,
                                 artifact_id TEXT NOT NULL, PRIMARY KEY(realization_id, name));
  INDEX idx_outputs_artifact(artifact_id)
CREATE TABLE source_observations(id TEXT PRIMARY KEY, artifact_id TEXT NOT NULL, provider TEXT NOT NULL,
                                 canonical_ref TEXT NOT NULL, record_json TEXT NOT NULL, observed_at TEXT NOT NULL);
CREATE TABLE attestations(id TEXT PRIMARY KEY, subject_artifact TEXT NOT NULL, predicate_type TEXT NOT NULL,
                          record_json TEXT NOT NULL, created_at TEXT NOT NULL);
CREATE TABLE checkpoints(id TEXT PRIMARY KEY, action_key TEXT NOT NULL, name TEXT NOT NULL,
                         artifact_id TEXT NOT NULL, record_json TEXT NOT NULL, created_at TEXT NOT NULL);
CREATE TABLE kv(key TEXT PRIMARY KEY, value TEXT NOT NULL);
```

Facts about how it is used:

- Every row is "a few indexed columns + the whole record as JSON" (`record_json`). Readers deserialize JSON (`metadata:259-272`). There are no `REFERENCES` clauses, so `foreign_keys=ON` enforces nothing today.
- No `synchronous` pragma is set, so the bundled SQLite default applies: `SQLITE_DEFAULT_SYNCHRONOUS 2` (FULL) and `SQLITE_DEFAULT_WAL_SYNCHRONOUS` = the same (checked in the local registry copy `libsqlite3-sys-0.28.0/sqlite3/sqlite3.c:17323-17326`; `Cargo.lock` pins 0.30.1, which was not in the local registry). No `busy_timeout` pragma either; rusqlite 0.32.1 installs a 5000 ms busy timeout on open (`rusqlite-0.32.1/src/inner_connection.rs:119`). This matches Flowyard's "WAL + synchronous=FULL" default without Artifactum having chosen it explicitly.
- Connection management: `MetadataStore { path: Arc<PathBuf>, conn: Arc<Mutex<Connection>> }` (`metadata:32-36`), one connection per open, `Clone` shares it, every method takes the `std::sync::Mutex` for the duration of one statement or one transaction (`metadata:61-63`). It is called from async code without `spawn_blocking`. Single writer within a process is therefore enforced by the mutex; across processes only by SQLite's file locks (the e2e cancel step runs `runs list` from a second process while a run is live, `scripts/e2e_observe.sh:187-197`).
- Transactions: only `record_realization` is a multi-statement transaction (`realizations` + `realization_outputs`, `metadata:137-153`). Everything else is a single `INSERT OR REPLACE`/`INSERT OR IGNORE` autocommit statement: `record_action` (`metadata:85-88`), `record_attempt` (`metadata:100-110`; `status` is derived at write time as `running`/`success`/`failed` from `finished_at` and `exit_code`), `record_source_observation` (`metadata:172-175`), `record_attestation` (`metadata:182-185`), `record_checkpoint` (`metadata:192-195`), `set_kv` (`metadata:206-209`). The engine's attempt-then-realization sequence is two separate commits with the executor call in between (`engine:317`, `engine:353`, `engine:416`).
- GC roots from metadata: realization outputs newer than `retention_days` (default 30 via `output_roots`, `metadata:240-242`), plus **all** source observations, checkpoints and attestation subjects (`metadata:216-239`). Refs and unexpired leases are added by the store (`crates/artifactum-store/src/lib.rs:676-681`).
- The connection is private: there is no `pub fn transaction(...)` or `pub fn connection()` on `MetadataStore`.

Could this file host Flowyard's run/operation/attempt tables? Yes, mechanically: `MetadataStore::open(path)` accepts any path (`metadata:43-56`), the evidence crate already places it at `<root>/metadata.sqlite` beside `<root>/store` (`evidence:592-597`), and the schema is created with `IF NOT EXISTS` so additional tables coexist. What it would buy: one durability domain and one file to back up; the ability to make Flowyard's "operation outcome" row and the Artifactum `realization` row commit in one SQLite transaction; a place to extend `gc_roots` so that artifacts referenced by an unfinished or retained Flowyard run are roots. What it does **not** give today: the same-transaction atomicity requires access to the single `Connection`, which `MetadataStore` does not expose, so either artifactum-metadata gains a transaction escape hatch (author owns both crates) or Flowyard opens a second connection to the same file, which gives shared durability but not cross-connection atomicity. Note also that the realization is already committed by the engine before `RunResult` returns, so a crash between "realization committed" and "Flowyard operation row committed" leaves a realization that the next run simply reuses (pure/reproducible) or re-executes (volatile/effect). The realization-first ordering already matches Flowyard's "commit bytes, then record the reference" rule (DESIGN.md section 3).

## Execution and executor

How an `ActionSpec` becomes an attempt (`Engine::run_inner`, `crates/artifactum-engine/src/lib.rs:158-430`):

1. Budget clamps timeout (`engine:164-167`); key computed and the spec upserted into `actions` (`engine:168-169`).
2. Cache check as described above (`engine:181-194`). No lock is taken on the action key: two concurrent `run` calls with the same key both miss and both execute (inference from the absence of any key-scoped lock; the only file lock in the workspace is the resolver's acquisition lock, `crates/artifactum-resolver/src/lib.rs:685`, `store:185-207`).
3. Executor resolved by name from a `HashMap<String, Arc<dyn Executor>>` (`engine:195-199`); the builder registers `local` by default and any `Executor + 'static` via `EngineBuilder::executor` (`engine:107-130`).
4. A store lease roots inputs and code against GC for `timeout + 600 s` (`engine:200-218`; lease files at `store:627-658`).
5. A `tempfile` sandbox with `in/`, `code/`, `out/`, `tmp/`, `checkpoint/in`, `checkpoint/out` (`engine:219-238`); env vars `ARTIFACTUM_ACTION_KEY`, `ARTIFACTUM_TMPDIR`, `ARTIFACTUM_OUT`, `ARTIFACTUM_CHECKPOINT_IN/OUT`, `ARTIFACTUM_INPUT_*`, `ARTIFACTUM_CODE_*`, `ARTIFACTUM_OUTPUT_*` (`engine:239-297`); inputs and code materialized read-only; `{in.X}`/`{code.X}`/`{out.X}` substituted in argv (`engine:298-302`).
6. **Admission commit:** `AttemptRecord` with `finished_at: None` is written (status `running`) before execution (`engine:303-317`). A cancel file path `<store>/control/<attempt>.cancel` is recorded in `kv` (`engine:318-325`).
7. Execution without any DB lock held (`engine:336`).
8. Executor error (timeout, unavailable, spawn failure): attempt recorded with `error`, checkpoint dir captured, lease released, `Err` returned (`engine:338-346`).
9. Otherwise: `finished_at`, `exit_code`, stdout/stderr as CAS `ContentId`s, metrics recorded (`engine:348-353`); checkpoint dir captured (`engine:354`); budget check (`engine:355-364`); non-zero exit becomes `ExecutionFailed { name, exit_code, attempt }` (`engine:365-373`); each declared output imported as blob/tree/collection or `MissingOutput` (`engine:374-394`); effect receipt (`engine:395-408`); **outcome commit** of the realization (`engine:409-416`).

Executor boundaries (`crates/artifactum-executor/src/lib.rs`): trait `Executor { name, execute(&ExecutionRequest) -> ExecutionResult, cancel (default no-op) }` (`executor:60-67`); `ExecutionRequest { action, command, cwd, env, resources, container, network, cancel_file }` (`executor:36-51`). Backends: `LocalExecutor` (`executor:69-86`), `BubblewrapExecutor` (`executor:88-124`, `--unshare-net` when network is deny/source-only), `ContainerExecutor` (`executor:126-169`), `SshExecutor` (rsync/scp round trip, `executor:171-311`), `SlurmExecutor` (`srun`, `executor:313-342`), `KubernetesExecutor` (`kubectl run/cp/exec`, `executor:344-473`), `PluginExecutor` (spawns `<exe> execute-json` with the request JSON on stdin, `executor:475-509`). Every one of them ends in `run_process` (`executor:511-594`), which spawns a `tokio::process::Command` with `kill_on_drop`.

- **Timeouts/budgets:** `resources.timeout_seconds` wraps the wait in `tokio::time::timeout`; on expiry the child is killed and `Error::Timeout` returned (`executor:551-562`). `budget.max_usd_micros` is compared against `cost_usd_micros_per_hour * wall` after the fact (`executor:574-578`, `engine:355-364`).
- **Cancellation:** `Engine::request_cancel(attempt)` writes the cancel file (`engine:612-628`); `run_process` polls for it every 150 ms, kills the child, and reports exit code 130 with a stderr note (`executor:546-550`, `executor:569-584`). The attempt is then recorded as failed via the non-zero-exit path. There is no attempt-authorization check on completion; in a single process nothing else can complete it, but nothing in the code expresses that.
- **Checkpoints:** `Checkpoint { id, action, name, artifact, created_at }` (`core:581-588`). Written by the action as files under `ARTIFACTUM_CHECKPOINT_OUT`; captured into the CAS after normal exit and after executor errors (`engine:531-543`, called at `engine:342` and `engine:354`); the newest checkpoint per name for the same action key is materialized read-only under `ARTIFACTUM_CHECKPOINT_IN` on the next attempt (`engine:251-261`, `metadata:200-205`). Granularity: per action key, named by file. The e2e script proves the fail-then-retry path (`scripts/e2e_observe.sh:146-170`).
- **Retry:** `Engine::retry_attempt(attempt)` loads the attempt, loads its action spec, and calls `run_uncached` with the same executor name (`engine:629-647`). There is no retry policy type, no attempt counter, no backoff, no `next_retry_at`; nothing about retry is persisted beyond the new attempt row. Attempt count per action key is derivable from `attempts_for_action` (`metadata:122-129`), but it is per key, not per run.
- **Determinism auditing:** `audit_determinism` reruns uncached N times and compares output ids (`engine:502-530`); `assert_deterministic_history` is enforced for `Pure` (`engine:490-501`).
- **stdout/stderr:** fully buffered in memory (`executor:538-545`), then stored as CAS blobs referenced from the attempt (`engine:350-351`); `artifactum runs logs <id> [--stderr]` prints them (`crates/artifactum-cli/src/main.rs:928-937`).
- **Process death mid-attempt:** nothing handles it. The attempt row stays `status='running'` with `finished_at NULL` forever; there is no sweep in `open`/`migrate` (`metadata:64-84`) or in the engine. The lease expires by TTL and is deleted on the next GC scan (`store:659-674`). The cancel file and the `/tmp/artifactum-run-*` sandbox are orphaned (the `TempDir` guard only runs on normal drop; inference). A rerun of the same spec re-executes for volatile/effect or, if an earlier realization exists, hits the cache.

**Is there an in-process Rust-function executor?** No. All seven backends spawn a process and the `ExecutionRequest` is argv/cwd/env-shaped (`executor:36-51`). Partial credit exists in two places: (1) the trait is object-safe and the builder accepts any implementation, so an in-process executor that interprets `command[0]` as an activity name, reads `ARTIFACTUM_INPUT_*` paths, writes to `ARTIFACTUM_OUTPUT_*` paths and returns an `ExecutionResult` would slot in without engine changes, at the cost of a filesystem round trip for every input and output and of re-implementing timeout and cancel-file polling that today live in `run_process`; (2) `Engine::realize_intrinsic(spec, outputs)` records an attempt and realization for a computation that already happened outside the engine (`engine:648-701`), which is how the evidence crate records externally executed runs (`evidence:859`). Neither gives typed inputs, application dependencies, or a cancellation token to a Rust function.

## Pipeline crate deep-dive

`crates/artifactum-pipeline/src/lib.rs` is the `Artifactum.toml` v3 model plus a runner.

- **Model:** `ProjectManifest { version, project, providers, remotes, artifacts, tasks, refs }` (`pipeline:153-169`); `TaskSpec { run, inputs, code, outputs, parameters, environment, resources, budget, cache, network, sandbox, executor, foreach }` (`pipeline:100-128`); `RefSpec { target, immutable }` (`pipeline:136-140`). Version 2 files are upgraded on load (`pipeline:187-210`).
- **Planning:** `plan()` computes the dependency closure of the targets (`pipeline:334-359`), then repeatedly peels every task whose task-dependencies are done into a level; an empty level means `Cycle` (`pipeline:295-333`). `Plan { levels: Vec<Vec<PlannedTask>> }` (`pipeline:284-287`).
- **Sources and locks:** `acquire_sources` resolves every `[artifacts.*]` through the resolver and upserts `Artifactum.lock` (`pipeline:488-510`, `pipeline:532-585`); `--frozen` requires the lock's requirement hash to match and the locked artifact to be loadable (`pipeline:553-570`).
- **Scheduling:** per level, a `Semaphore(max_parallel)` and a `JoinSet` (`pipeline:439-467`); `max_parallel` defaults to 8 and the CLI passes `--jobs` (`pipeline:417`, `cli:42`, `cli:533`). Inside a `foreach` task, a second `Semaphore(max_parallel)` bounds items (`pipeline:649-679`). Outputs are accumulated in an in-memory `Mutex<BTreeMap<"task.output", ArtifactId>>` (`pipeline:433-435`).
- **foreach:** the target must be a collection or a tree; tree blobs are lifted into per-file blob artifacts keyed by path (`pipeline:616-648`). Each item becomes its own `ActionSpec` with input `item`, name `task[key]`, and parameter `artifactum_map_key = key` (`pipeline:664-675`, `pipeline:786-801`), so **one item is one action identity** and unchanged items are cache hits (`scripts/e2e_observe.sh:111-115` asserts 2 hits + 2 misses after one input changes). After all items complete, per-output collections are built and recorded as an intrinsic `Pure` action with argv `["artifactum:collect"]` whose inputs are the item outputs keyed by item key (`pipeline:696-730`). Ordering is by sorted key (`CollectionManifest::new` sorts, `core:293-302`), not by completion order.
- **Failure policy:** none configurable. The first failing task in a level (`r??` at `pipeline:465`) or the first failing item (`pipeline:683`) returns `Err`; dropping the `JoinSet` aborts the sibling tasks and `kill_on_drop` kills their children (inference from Tokio semantics; the code does not handle partial success). Successful sibling realizations are already committed and will be cache hits next time.
- **Publication:** after all levels, every `[refs.*]` is resolved and `store.set_ref(name, id, immutable)` is called (`pipeline:469-472`; immutable refs refuse overwrite, `store:578-594`). Refs are JSON files under `<store>/refs/`, written atomically per file but not transactionally with each other or with the SQLite plane.
- **Run representation:** `PipelineRun { sources, outputs, actions: BTreeMap<task, Vec<RunResult>> }` (`pipeline:288-293`) is returned to the caller and printed by the CLI (`cli:534-535`). **It is never persisted.** There is no run table, no run id, no per-task progress record. Durable state during a pipeline run consists only of the engine's per-action rows and the lockfile.
- **Process death mid-pipeline:** a rerun re-resolves sources (or uses `--frozen`), re-plans, and calls `engine.run` for every action again; completed pure/reproducible actions hit the cache, everything else re-executes. Resume is therefore "recompute the DAG against the action cache", exactly the Prefect/Dagster-style reuse that DESIGN.md section 1 says Artifactum should own, and not a journal.

Contrast with Flowyard's named-step model: the pipeline crate is a declarative, command-oriented, level-parallel DAG executor in the Dagu family (DESIGN.md section 1, "Dagu and Scrapy"). Its identities are action keys derived from content, not operation keys scoped to a run; it has no notion of a run, an in-progress operation, a retry budget, a child relationship, or a recovery policy. It competes with a Rust-function workflow API only at the level of "how do I express a two-step transform over a directory", and Flowyard's `map` over a keyed collection can reuse its per-item action identity and intrinsic collection realization directly. It is complementary: Flowyard would call `Engine::run`/`realize_intrinsic` per activity and would not need the TOML planner at all.

## Receipts and evidence

- `artifactum-receipt` defines `ReceiptEnvelope<P> { schema, receipt_id, producer, activity, environment, inputs, outputs, command, diagnostics, started_at, finished_at, payload }` (`crates/artifactum-receipt/src/lib.rs:178-195`) with `ProducerIdentity` (`receipt:54-60`) and `ActivityIdentity { action: ActionKey, attempt, parent: Option<ReceiptId> }` (`receipt:79-85`). The receipt id is SHA-256 over everything except the id itself (`receipt:201-235`) and `validate` recomputes it (`receipt:237-251`). It is a contract crate with no I/O.
- `artifactum-evidence` builds on the engine: `EvidenceStore::open(root)` opens `<root>/store` and `<root>/metadata.sqlite` (`evidence:592-597`); `put_asset`/`put_asset_file` refuse bytes whose producer-declared digest (sha256 or blake3) does not match before storing anything (`evidence:627-693`, tested `tests/evidence_roundtrip.rs`); `record_run` derives an `ActionSpec` from the description with `cache = Effect` so it is never replayed (`evidence:729-771`), records it via `realize_intrinsic` (`evidence:859-865`), and writes `produced-by`/`consumed-by`/`executed-by` attestations as the reverse index and GC roots (`evidence:876-895`, `docs/EVIDENCE.md:73-83`); `record_claim` seals a record snapshotting every cited digest (`evidence:1005`); `verify_claim` re-hashes everything and reports failures by path (`evidence:1209`); `explain` walks the attestations (`evidence:1404`).
- Mapping onto Flowyard: the engine's effect receipt (`engine:395-408`) and the evidence run collection are both "an immutable record that an effect happened", which is Flowyard's **effect** outcome. Source observations (`core:334-351`, written by the resolver at `resolver:619-646`) are Flowyard's **observations**: same bytes can be observed from many providers without changing identity. Neither the receipt nor the effect receipt carries an **external idempotency key**; `ActivityIdentity.attempt` is a free string and `payload` is opaque, so Flowyard would have to put its idempotency key into `spec.parameters` (which enters the action key, `core:506`) or into the receipt payload.

## Coverage matrix against Flowyard's Keep list

DESIGN.md:440 lists the Keep items. Gap sizes: small = days inside existing types, medium = a new module with its own tables/state machine, large = a new subsystem.

| Keep item | Artifactum has | Missing | Gap |
|---|---|---|---|
| Typed workflows and activities | `ActionSpec` builder (`action:20-101`); typed `OutputSpec` kinds; JSON `parameters` (`core:457`) | Any notion of a Rust function as an activity, input/output codecs, a workflow definition, registration, macros (no proc-macro crate in the workspace) | large |
| Named operation identities | Content-derived `ActionKey` (`core:495-525`); per-item map key in name and parameters (`pipeline:786-801`); task-name index in `kv` (`engine:170-180`) | Run-scoped operation keys, operation type/contract validation on re-entry, mismatch errors | medium |
| One SQLite backend | WAL SQLite plane with 8 tables (`metadata:64-84`); path-configurable; effective FULL sync | Run/operation/attempt/schedule/child tables; transaction escape hatch; foreign keys; a restart sweep | medium |
| Crash recovery | Attempt committed before execution (`engine:317`), realization after (`engine:416`); checkpoints re-materialized on retry (`engine:251-261`); cache reuse on rerun (`engine:181-194`) | Orchestration re-entry, per-operation recovery policy (repeat-safe/idempotent/uncertain), orphaned `running` attempt handling, workspace exclusive lock | large |
| Durable retries and deadlines | `retry_attempt` re-runs uncached (`engine:629-647`); per-attempt timeout (`executor:551-562`); attempt history per key (`metadata:122-129`) | Retry policy, attempt count per operation, persisted `next_retry_at` with jitter, transient/terminal/reconcile classification, durable timers | medium |
| Structured keyed children | `foreach` = one action per item key, deterministic keyed collection realization (`pipeline:614-730`); `CollectionManifest` sorted by key (`core:293-302`) | Recorded work set per run, unique-key validation, per-child recovery, parent-child rows committed together, completion policies | medium |
| Cancellation state | Cancel file per attempt, polled at 150 ms, exit 130 (`engine:612-628`, `executor:546-584`); CLI `runs cancel` (`cli:943-947`) | Persisted cancellation state on operations/runs, cancellation token for in-process work, "attempt no longer authorized" check at completion | medium |
| Artifactum integration | Everything: CAS, manifests, refs, leases, GC, lineage, receipts | The bridge itself (typed value to artifact and back, in-process executor or intrinsic realization path, GC roots for Flowyard references) | medium |
| Source-level admission controls | Global `--jobs` semaphore per level and per foreach (`pipeline:440`, `pipeline:649`); resolver `max_concurrency`, default 8 (`resolver:437`, `resolver:585`) | Named concurrency limits, per-host cooldowns, server-requested cooldown persistence (ROADMAP.md:20 lists per-origin limiting as not done) | medium |
| Inspection | `status`, `runs list/show/logs/retry/cancel`, `lineage`, `why` with structural diff, `audit determinism`, `checkpoint get/put` (`cli:50-150`, `cli:918-975`); JSON output everywhere | `describe`, `inspect <run>`, `trace <run>`, `explain-cache <operation>`; anything run- or operation-scoped | medium |

## Answers to Flowyard's open questions where Artifactum's code gives evidence

- **Q1 unreached recorded operations:** no evidence. There is no run or operation model; actions recorded in `actions` but never executed simply have no attempts.
- **Q2 singleton per key:** Artifactum does not enforce it. `run_inner` takes no per-key lock (`engine:158-430`); the only key-scoped lock is the resolver's acquisition lock for downloads (`resolver:685`, `store:185-207`, a `create_new` file spun for up to 60 s). Store leases are GC roots, not mutual exclusion (`store:627-674`). Two concurrent runs of one pure key would both execute and the second realization would then trip the determinism check only if outputs differ.
- **Q3 daemon vs one-shot:** the engine and CLI are one-shot per invocation; the only daemon is the provider plugin host, which survives across CLI calls and is respawned after being killed (`scripts/e2e_observe.sh:293-313`). Nothing about execution state assumes a resident process.
- **Q4 inline vs queued execution:** inline. `Engine::run` executes in the caller's task; the pipeline fans out with `JoinSet` + `Semaphore` (`pipeline:439-467`, `pipeline:649-679`). Nothing is queued durably; a queued action has no row.
- **Q5 per-host cooldowns:** none. Concurrency is one global `--jobs` bound; `ROADMAP.md:20` lists "per-origin concurrency/rate limiting" as future work.
- **Q6 checkpoint granularity:** per action key, named files written by the action process itself (`engine:251-261`, `engine:531-559`); the newest per name wins (`metadata:200-205`). There is no orchestration-level checkpoint.
- **Q7 large outputs inline vs referenced:** always referenced. Outputs, stdout and stderr are CAS ids (`realization_outputs`, `metadata:73`; `engine:350-351`); only `parameters` JSON is inline in `spec_json` and it enters the action key (`core:506`). Small typed activity results would need a new convention (blob artifact of canonical JSON is the cheapest fit; `ArtifactStore::artifact_from_bytes` exists, `store:409`).
- **Q8 code-change gate:** the action key includes whatever code artifacts the caller declares (`core:505`), and `why` diffs old vs new spec (`engine:468-489`, `cli:951-975`). Nothing hashes the host binary, and nothing rejects continuation; the receipt's `ProducerIdentity { repository, commit, package_version, executable }` (`receipt:54-60`) is the only existing build-identity record and it is informational.
- **Q9 retry persistence:** only attempt rows persist (`metadata:100-110`). No policy, no counter, no scheduled time; `retry_attempt` is "run uncached now" (`engine:629-647`). Checkpoint artifacts do persist and are the one retry-relevant state that survives a restart.
- **Q10 missed scheduled runs:** no evidence; there is no scheduler, timer or schedule table anywhere in the workspace.

## What "a nice macro system on top" would actually have to generate

Given the existing types, a `#[flowyard::activity]` / `#[flowyard::workflow]` layer would need to produce:

1. **Input/output codecs to artifacts.** Serialize typed inputs to canonical bytes and store them via `ContentStore::put_bytes` + `ArtifactManifest` (`store:103-109`, `core:305-332`) or `artifact_from_bytes` (`store:409`), producing the `ArtifactId`s that go into `spec.inputs`; deserialize outputs from `Realization.outputs`. Hook exists (store API); nothing typed exists. For large outputs the existing `OutputSpec { kind: Tree | Collection }` and `import_tree`/`put_collection` (`store:347-391`) are the hooks.
2. **`ActionSpec` construction.** `ActionBuilder` (`action:20-101`) covers argv, inputs, code, outputs, parameters, env, policies. For a Rust activity the argv would be a synthetic marker such as `["flowyard:activity", name, version]`; `parameters` carries the small typed inputs and the external idempotency key. Hook exists; the convention does not.
3. **Code identity.** Something to put into `spec.code` (`core:455`): a build-digest blob (binary hash, or cargo package version + git commit in the `ProducerIdentity` shape, `receipt:54-60`). No hook computes this; the macro or build script must supply it, and DESIGN.md section 4 wants a conservative build digest rather than a function-body hash.
4. **Registration.** A static registry `name -> (input codec, output codec, boxed async fn, policy)`. No hook; Artifactum's only registry is `EngineBuilder::executor` keyed by executor name (`engine:127-130`).
5. **Invocation adapter.** Either (a) an `Executor` implementation that dispatches on `command[0]`, reads/writes the sandbox paths from `ExecutionRequest.env`, and re-implements timeout and cancel-file polling (`executor:60-67`, `executor:546-562`); or (b) a direct path that computes `spec.key()`, checks `metadata.latest_realization` + `realization_available` for pure/reproducible (`engine:181-194` logic, not exposed as a function), runs the function in-process with a `CancellationToken` and `tokio::time::timeout`, imports outputs, and calls `Engine::realize_intrinsic` (`engine:648-701`). Path (b) skips materialization for small values but must reproduce the attempt-before/realization-after commits itself because `realize_intrinsic` writes a finished attempt and the realization together (`engine:665-692`).
6. **Workflow context plumbing.** `WorkflowContext::call(key, Activity, input)` must map to: operation row (Flowyard), then activity invocation (item 5), then outcome row referencing `RunResult { action, attempt, realization, cache_hit }` (`engine:65-71`). No hook in Artifactum.
7. **Definition metadata for tooling** (`describe`, generated CLI args, schemas): nothing exists; `ActionSpec` is the only descriptor and it is per-invocation, not per-definition.

Items 1, 2 and 5(a)/(b) have hooks; 3, 4, 6 and 7 do not.

## Risks and invariants

Invariants a Flowyard layer must not violate (`AGENTS.md:3-11`, `docs/ARCHITECTURE.md:5-16`):

- Keep `ContentId` / `ArtifactId` / `ActionKey` distinct; never derive one from another's provenance.
- Never put credentials or signed URLs into semantic identity; an idempotency key or a bearer token in `parameters` would enter the action key (`core:506`), so Flowyard's external idempotency key must go there deliberately and secrets must not.
- Never turn a failed attempt into a realization (`engine:365-373` enforces it; a direct `realize_intrinsic` caller can bypass it, `engine:648-701`).
- Never cache-hit `volatile` or `effect` (`engine:181-183`); a Flowyard "observation" activity should be `Volatile` or `Effect`, never `Reproducible`, or a second run will silently return stale bytes.
- Keep scheduling-only fields (retry count, concurrency, deadlines, run id) out of `ActionKey`; put them in Flowyard's tables, not in `parameters`.
- GC honors refs, unexpired leases, recent realizations (30-day default), source observations, checkpoints and attestations (`metadata:216-239`, `store:676-681`). A Flowyard run older than the retention window that references artifacts only through its own tables would lose them unless Flowyard passes `extra_roots` to `gc` (`store:726`) or writes attestations/refs. DESIGN.md section 3 requires exactly this awareness.

Places that look unfinished or untested:

- Engine, executor, pipeline, action and CLI have no unit tests; coverage is the e2e script only (`scripts/e2e_observe.sh`), which never kills the CLI mid-attempt (its only kill is the provider daemon, `e2e:291-313`) and never exercises `--frozen` failure, budget rejection, ssh/slurm/k8s/container executors, or concurrent writers.
- `record_attempt` derives `status` at write time and nothing ever revisits `running` rows (`metadata:100-110`); `Status` and `runs list` will show phantom running attempts after a crash.
- Metadata `Mutex<Connection>` is held on the async runtime thread; long transactions would stall the runtime (inference).
- `import_collection_dir` errors on an empty output collection (`engine:460-462`), so an activity that legitimately produces zero items cannot use a `Collection` output.
- The `TempDir` sandbox lives in the system temp dir, not the store (`engine:219-222`); outputs are imported by copy, and orphaned sandboxes after a crash are never cleaned.
- Refs are per-file JSON, not in SQLite (`store:578-594`), so Flowyard's "one authoritative pointer updated in the same transaction as the publication operation" (DESIGN.md section 6) cannot use Artifactum refs for the transactional part.
- `GENERATION_NOTES.md:15-18` states the workspace was generated without a compiler; later commits made it build, and `STATUS.md:96-101` records `cargo test --workspace` at 38 passed but the e2e script "not run" for the evidence lane. No date-stamped record shows the e2e script passing at HEAD.
- No `TODO`/`FIXME` markers exist in non-provider crates (grep count 0), which says nothing about completeness.

ROADMAP overlap: per-origin concurrency and rate limiting (`docs/ROADMAP.md:20`), timeout/retry policy objects (`ROADMAP.md:21`), and process-crash resume journals for HTTP acquisition (`ROADMAP.md:19`) are all listed as future Artifactum work and overlap Flowyard's "durable retries", "source-level admission controls" and "crash recovery" items. Nothing in the roadmap mentions runs, workflows, schedules or an in-process executor.

## Recommendation

**(a) with one amendment:** put Flowyard's run/operation/attempt/schedule tables in the same SQLite file that `artifactum-metadata` opens, and drive `artifactum-engine` (or its store + `realize_intrinsic` path) for activities, but add a small transaction escape hatch to `artifactum-metadata` so both planes can commit together. Reasons anchored in the code:

- The file is already path-configurable and multi-purpose (`metadata:43-56`; evidence crate places it beside the CAS, `evidence:592-597`), and its pragmas already match Flowyard's stated defaults (WAL, effective FULL sync, 5 s busy timeout).
- The engine's commit ordering (attempt before execution `engine:317`, realization after `engine:416`, no transaction held across execution `engine:336`) is the same shape as Flowyard's commit-boundary rule, so Flowyard's operation rows can bracket a single `Engine::run` call without fighting it.
- GC correctness (DESIGN.md section 3) is only achievable if `gc_roots` can see Flowyard's references; that is a one-function change in the same crate (`metadata:216-239`), whereas a separate file forces Flowyard to re-implement root discovery through `extra_roots` on every GC call.
- Keeping a separate Flowyard file (option b) would buy independence from Artifactum's schema at the cost of two durability domains, no cross-plane atomicity, and a permanent risk of dangling artifact references after GC. Nothing in the read code suggests Artifactum's schema is unstable enough to justify that.

The two or three largest pieces of new code either way:

1. **The workflow journal and re-entry engine:** runs, operations, attempts, children, schedules; exclusive workspace lock; restart sweep of `running` attempts; operation-key validation on re-entry; recovery policy per operation (repeat-safe / idempotent / uncertain); persisted retry schedule. Nothing in Artifactum covers any of it.
2. **The in-process activity boundary:** registry, codecs between typed values and artifacts, cancellation token and timeout, code/build identity, and either an `Executor` implementation or a direct cache-check + `realize_intrinsic` path. Artifactum supplies the storage and identity calls but no function-shaped execution.
3. **The macro and tooling layer:** `#[activity]`/`#[workflow]` generation, definition metadata, and run-scoped `inspect`/`trace`/`explain-cache` commands; the existing CLI `runs`/`why`/`lineage` commands (`cli:918-975`) are per-attempt and per-action and can be reused underneath.

## Sources

Files read in full unless a range is given.

| path | lines |
|---|---|
| /projects/flowyard/docs/DESIGN.md | 454 |
| README.md | 195 |
| AGENTS.md | 15 |
| STATUS.md | 106 |
| GENERATION_NOTES.md | 36 |
| AGENT_TESTING.md | 1-50, plus grep for crash/kill/restart |
| docs/ARCHITECTURE.md | 101 |
| docs/EXECUTION.md | 38 |
| docs/STORE_V2.md | 36 |
| docs/PROJECT_FORMAT.md | 60 |
| docs/ROADMAP.md | 108 |
| docs/EVIDENCE.md | 162 |
| crates/artifactum-core/src/lib.rs | 659 |
| crates/artifactum-metadata/src/lib.rs | 309 |
| crates/artifactum-action/src/lib.rs | 161 |
| crates/artifactum-engine/src/lib.rs | 747 |
| crates/artifactum-executor/src/lib.rs | 635 |
| crates/artifactum-pipeline/src/lib.rs | 853 |
| crates/artifactum-store/src/lib.rs | 87-140, 185-207, 578-760, test names 1020-1125; pub surface grep |
| crates/artifactum-receipt/src/lib.rs | 379 |
| crates/artifactum-evidence/src/lib.rs | 585-626, 729-800, 845-880; pub surface grep |
| crates/artifactum-resolver/src/lib.rs | `ArtifactProvider` trait 288-350, `acquire_resolution` body; grep |
| crates/artifactum-provider-api/src/lib.rs | 60 |
| crates/artifactum-provider-sdk/src/lib.rs | pub surface grep |
| crates/artifactum-cli/src/main.rs | 29-50, 519-600, 918-975; subcommand grep |
| crates/artifactum-metadata/Cargo.toml, Cargo.toml (workspace), Cargo.lock (rusqlite/libsqlite3-sys pins) | |
| scripts/e2e_observe.sh | 1-215, 280-349, plus grep of step markers |
| ~/.cargo/registry/.../rusqlite-0.32.1/src/inner_connection.rs:119; libsqlite3-sys-0.28.0/sqlite3/sqlite3.c:17303-17326 | |
