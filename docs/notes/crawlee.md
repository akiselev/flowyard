# Crawlee (JavaScript and Python)

Researched: 2026-09-16. Scope: Crawlee for JavaScript 3.18.1 (npm `latest`, 2026-08-12) with notes on the 4.0.0 line (npm `rc` 4.0.0-rc.0; GitHub master `packages/crawlee` 4.0.0); Crawlee for Python 1.10.1 (PyPI). Docs at crawlee.dev (JS "current" and "next", Python), source from GitHub `apify/crawlee` (master and tag v3.15.0) and `apify/crawlee-python` (master).

## Summary
Crawlee is Apify's crawling library: a `RequestQueue` (persisted, deduplicated by `uniqueKey`), a router that dispatches each request to a labelled handler, an `AutoscaledPool` that runs handlers concurrently under CPU/memory pressure limits, a `SessionPool` for cookies/proxies, and `Dataset`/`KeyValueStore` for results. It runs inside your process; there is no server. Its durability unit is the request: requests survive restarts, handlers are re-run from scratch, and retry counts travel with the request. It is the ergonomics benchmark for scraping because a complete crawler is ~10 lines and routing by label scales cleanly.

## Deployment and process model
- Library (Node.js or Python). `crawler.run(urls)` blocks until the request manager reports finished (`keepAlive` keeps polling instead). One process; the pool is in-process.
- Storage is pluggable via a storage client: JS 3.x `MemoryStorage` (in-memory + optional disk persistence under `CRAWLEE_STORAGE_DIR`, default `./storage`); JS 4.x `FileSystemStorageBackend` "backed by the native `@crawlee/fs-storage-native` Rust extension" (repo `apify/crawlee-storage`); Python `FileSystemStorageClient`, `MemoryStorageClient`, `SqlStorageClient` (SQLite by default: "creates an SQLite database file named `crawlee.db` in the storage directory"; Postgres/MySQL via connection string), `RedisStorageClient`, `ApifyStorageClient`.
- Multi-process sharing of one on-disk queue is opt-in in JS 4 (`requestQueueAccess: 'shared'`), lease-based in the Python SQL client (`time_blocked_until`, `client_key`, `skip_locked` on Postgres/MySQL).

## Execution model
- Explicit persisted structure (the queue), no replay, no step checkpointing. A handler is an ordinary async function; nothing it does is recorded except `pushData` to the append-only Dataset and whatever it writes to the KVS.
- Resuming: JS `BasicCrawler` docs/source — calling `run()` on an instance with existing requests in the manager continues; already-handled requests are not reprocessed. Python `run()` docstring: "purge_request_queue: If this is `True` and the crawler is not being run for the first time, the request queue will be purged. A run that ended with an exception does not count as a previous run, so a retry keeps the requests that were still pending. Named request queues are considered persistent and are never purged implicitly."
- The default (unnamed, "run-scoped") storages are purged at start unless `purgeOnStart: false` / `CRAWLEE_PURGE_ON_START=false` (JS `ConfigurationOptions.purgeOnStart` default `true`: "Defines whether to purge the default storage folders before starting the crawler run.") or `Configuration(purge_on_start=False)` in Python. JS "next" docs: "Run-scoped storages - the default one and any opened with an alias - are purged before the crawler starts if not specified otherwise." "Storages opened with a name persist across runs and are never purged."
- Python example page "Resuming a paused crawl": "Disable clearing the `RequestQueue`, `KeyValueStore` and `Dataset` on each run. This makes the scraper continue from where it left off in the previous run."
- No determinism rules; handlers may do anything, and a handler interrupted by a crash is simply run again.

## Step and operation identity
- `Request.uniqueKey`: "Two requests with the same `uniqueKey` are considered as pointing to the same web resource." Default: "normalized URL, lowercased, without fragment"; `keepUrlFragment` keeps it; `useExtendedUniqueKey` adds method, payload and headers to the hash. RequestQueue docs: "The queue can only contain unique URLs. More precisely, it can only contain Request instances with distinct `uniqueKey` properties." To add the same URL twice, supply a different `uniqueKey`.
- Request id: JS memory-storage derives a UUID-like id from `uniqueKey`; Python derives `request_id` from `sha256(unique_key)` truncated to 15 hex chars (FS client file name and SQL client `request_id`).
- `label` (shortcut for `userData.label`) selects the handler via `Router`; `userData` is arbitrary JSON carried with the request.
- Dedup result on add: `wasAlreadyPresent` / `wasAlreadyHandled` per request (JS `addBatchOfRequests`, Python `AddRequestsResponse`). Adding an already-handled key is a no-op; there is no "same key, different input" detection beyond the key itself.
- Unreached/mismatch: n/a (no replay). A request whose label has no handler goes to the default handler or errors (Router behavior).

## Persistence schema
JS 3.x (docs `guides/request-storage`, `guides/result-storage`):
- `{CRAWLEE_STORAGE_DIR}/request_queues/{QUEUE_ID}/entries.json` ("entries.json contains an array of requests"); `{CRAWLEE_STORAGE_DIR}/key_value_stores/{STORE_ID}/{KEY}.{EXT}`; `{CRAWLEE_STORAGE_DIR}/datasets/{DATASET_ID}/{INDEX}.json`. Default ids are `default` (override with `CRAWLEE_DEFAULT_REQUEST_QUEUE_ID` etc.).
- Source (tag v3.15.0 `memory-storage/.../request-queue.ts`): `InternalRequest { id, orderNo: number | null, url, uniqueKey, method, retryCount, json }`; `orderNo` = `null` when handled, `-Date.now()` for forefront, `Date.now()` otherwise; lock = `orderNo` moved to a future timestamp (`Math.abs(orderNo) + lockSecs * 1000`); `listAndLockHead`, `prolongRequestLock`, `deleteRequestLock`; entries persisted per request through `RequestQueueFileSystemEntry` and metadata written by a background `update-metadata` task (so the docs' `entries.json` wording and the source's per-entry storage should both be checked against the version you run).
- Dataset: "Dataset is an append-only storage - we can only add new records to it, but we cannot modify or remove existing records."
JS 4.x (`packages/fs-storage/src/file-system-storage.ts`): subdirectories `datasets/`, `key_value_stores/`, `request_queues/` under `localDataDirectory`; format owned by the native crate.
Python FS client (`_file_system/_request_queue_client.py`): "Each request is stored as a separate file in a directory structure following the pattern: `{STORAGE_DIR}/request_queues/{QUEUE_ID}/{REQUEST_ID}.json`" plus `metadata.json`; ordering/in-progress/handled state kept in a `RequestQueueState` (`sequence_counter`, `forefront_sequence_counter`, `forefront_requests`, `regular_requests`, `in_progress_requests`, `handled_requests`) persisted through `RecoverableState` into a KeyValueStore, "without embedding metadata in individual request files". Writes use `atomic_write`; all operations hold an `asyncio.Lock`.
Python SQL client (`_sql/_db_models.py`): `request_queues` (metadata, counts, `had_multiple_clients`, `buffer_locked_until`), `request_queue_records` (`request_id` BigInteger + `request_queue_id` PK, `sequence_number`, `is_handled`, `time_blocked_until`, `client_key`, `data` Text; indexes `idx_fetch_available (request_queue_id, is_handled, sequence_number)`), `request_queue_state` (`sequence_counter`, `forefront_sequence_counter`), `datasets`/`dataset_records` (`data` JSON), `key_value_stores`/`key_value_store_records` (`key`, `value` LargeBinary, `content_type`, `size`), `version`.
- Large outputs: Dataset items are whole JSON files/rows; KVS values are files with a MIME type (screenshots, PDFs) or `LargeBinary` rows. Nothing is inlined into a request record beyond `userData`.

## Crash recovery of in-flight work
Per storage client (from the sources cited in Persistence schema):

| Client | On-disk unit | In-progress marker | After a crash |
|---|---|---|---|
| JS 3.x MemoryStorage (`persistStorage`) | per-request entry + metadata | `orderNo` pushed to `now + lockSecs` | request reappears when the timestamp lock expires |
| JS 4.x FileSystemStorageBackend (native) | request_queues/ dir, native format | native state file, persisted on `teardown()` | `'single'`: reclaimed immediately on open; `'shared'`: after wall-clock lock expiry |
| Python FileSystemStorageClient | `{REQUEST_ID}.json` + `metadata.json` + RecoverableState in KVS | `in_progress_requests` set (state only) | `_discover_existing_requests` clears the set: all in-progress become pending |
| Python SqlStorageClient (SQLite/PG/MySQL) | `request_queue_records` rows | `time_blocked_until` + `client_key` | fetchable again after `_BLOCK_REQUEST_TIME = 300` s |
| Python MemoryStorageClient | none | in memory | everything lost |

- JS 4 `requestQueueAccess` (verbatim): "With `'single'` (the default), this process asserts it is the *sole* consumer of every request queue it opens: on open, any requests that a previous run left *in progress* (e.g. after a crash) are reclaimed immediately, so they become fetchable again right away. This is the right behavior for the common single-process crawl." "Use `'shared'` if multiple processes share the same on-disk request queue concurrently ... In that mode an in-progress request is treated as a potential live peer's lock and is only reclaimed once that lock expires on the wall clock, so two workers won't process the same request at once." `teardown()` "persists the state of every opened request queue so that requests fetched but not yet handled are not stuck (until their lock expires)".
- JS 3 (memory-storage): the lock is the `orderNo` timestamp; after a crash a request stays "locked" until `lockSecs` elapses, then is listed again (inferred from the lock representation; there is no explicit reclaim-on-open in that file).
- Python FS client `_discover_existing_requests`: "On recovery after a crash, any requests that were previously in-progress are reclaimed as pending, since there is no active processing after a restart." Logs "Reclaiming N in-progress request(s) from previous run." Handled requests are kept "only for deduplication, not as pending work".
- Python SQL client: `_BLOCK_REQUEST_TIME = 300` seconds; `fetch_next_request` takes rows with `is_handled == False` and `time_blocked_until` null or expired, sets `time_blocked_until` and `client_key`; a dead client's rows become fetchable after 5 minutes.
- What the user can declare: nothing per request about idempotency; a request re-run after a crash re-executes the whole handler, so `pushData` from the first partial run may be duplicated in the Dataset (append-only). `noRetry` and `maxRetries`/`max_request_retries` govern handler errors, not crash re-runs (`retryCount` is only incremented on a handled error).

## Retries, backoff, timeouts
- `maxRequestRetries` default 3 (JS source `schemas.anyNumber.default(3)`; Python `max_request_retries: int = 3`): "Specifies the maximum number of retries allowed for a request if its processing fails." Session rotations are separate: `maxSessionRotations` default 10 (JS), `max_session_rotations: int = 10` (Python), "Not counted toward maxRequestRetries limit".
- Flow (JS `requestFunctionErrorHandler`, Python `_handle_request_retries`): if `request.noRetry` is false and `retryCount < maxRequestRetries`: increment `retryCount`, call `errorHandler` ("allows modifying the request object before it gets retried"), then `reclaimRequest()` (back to the queue; `forefront` optional). Otherwise mark handled, set error state, and call `failedRequestHandler` ("A function to handle requests that failed more than maxRequestRetries times ... Second argument is the Error instance that represents the last error thrown"). Python also has `abort_on_error=False`.
- Persistence: `retryCount` is a field of the stored request (JS `InternalRequest.retryCount`, Python `Request.retry_count`), so the allowance survives restart. No next-attempt time and no backoff between retries are documented; a reclaimed request is immediately eligible (unverified whether any implicit delay exists).
- Per-request overrides: `noRetry`, `maxRetries` (JS Request option "Override global retry limit"), `skipNavigation`.
- Timeouts: `requestHandlerTimeoutSecs` default 60 (JS), `request_handler_timeout = timedelta(minutes=1)` (Python); a timeout is an error and follows the retry path. `retryOnBlocked` (JS `true`? documented as "If set to true..."; Python `retry_on_blocked: bool = True`) retries on detected bot protection.
- Failure classification: none built in beyond blocked-detection and session errors (`session.markBad()` on network errors, `session.retire()` on confirmed blocks); everything thrown is retryable until the count runs out.

## Flow control
- `AutoscaledPool` (JS docs/defaults): `minConcurrency` 1, `maxConcurrency` 200, `desiredConcurrency` = `minConcurrency`, `desiredConcurrencyRatio` 0.90 ("Minimum level of desired concurrency to reach before more scaling up is allowed"), `scaleUpStepRatio`/`scaleDownStepRatio` 0.05, `autoscaleIntervalSecs` 10, `maybeRunIntervalSecs` 0.5, `taskTimeoutSecs` 0, `maxTasksPerMinute` Infinity; scaling is driven by `Snapshotter`/`SystemStatus` (CPU, memory via `CRAWLEE_MEMORY_MBYTES`/`availableMemoryRatio` 0.25, event-loop lag). Python: `ConcurrencySettings(min_concurrency, max_concurrency, desired_concurrency, max_tasks_per_minute)`.
- Rate: `maxRequestsPerMinute` ("By default, this is set to Infinity") counted per second to avoid bursts. `maxRequestsPerCrawl` stops the crawl (may overshoot slightly in parallel). `sameDomainDelaySecs` default 0 (JS): master wraps the manager in a `ThrottlingRequestManager` with `throttleBy: 'registrableDomain'`. Python: `max_crawl_depth`.
- Keyed by: process (pool), registrable domain (`sameDomainDelaySecs`). All in-memory, per process, not persisted.
- `SessionPool`: `maxPoolSize` default 1000; `maxUsageCount`, `maxErrorScore`, `maxAgeSecs`, `blockedStatusCodes`; state persisted to the KVS under `SDK_SESSION_POOL_STATE` on the `persistState` event (`persistStateIntervalMillis` default 60_000). This is the one piece of politeness state that survives a restart.

## Fan-out, child workflows, map
- `enqueueLinks({ selector, label, ... })` / `context.enqueue_links()` and `addRequests` push into the same queue; there is no parent/child or collect primitive. The "work set" is discovered incrementally; `RequestList` (JS) or `add_requests` with a fixed list gives an up-front set, but completion is still just "queue finished".
- Results: `Dataset.pushData`; ordering = `{INDEX}.json` insertion order; failure policy = `failedRequestHandler` per request; nothing aggregates outcomes per source.
- `useState()` / `use_state()` gives an auto-persisted KVS value (JS warns when multiple crawlers share it without an id; Python key `CRAWLEE_STATE_<crawler_id>`).

## Versioning and code change policy
- None. Handlers are looked up by label at runtime; changing code between runs is invisible to the queue. No version stored with a request.

## Schedules, timers, missed runs
- None in the library (scheduling is an Apify-platform feature). `keepAlive` keeps the crawler polling an empty queue so an external producer can feed it. No durable timers.

## Cancellation and late results
- `crawler.stop()` / `teardown()`; `isTaskReadyFunction` returns false after `stop()`, `isFinishedFunction` waits for "remaining requests [to] finish". Handler timeouts abort the task and enter the retry path. A request finishing after teardown in JS 4 'single' mode would be reclaimed and re-run on next open (inferred from the reclaim-on-open rule).

## Observability and tooling
- `Statistics` persisted to the KVS (JS `statisticsOptions`, Python `Statistics.with_default_state(persistence_enabled=True, ...)`); periodic status messages (`statusMessageLoggingInterval`, Python `status_message_logging_interval = 10s`); `log` with `CRAWLEE_LOG_LEVEL`; `getInfo()` on queues (handled/pending counts). `crawlee` CLI scaffolds projects. No run-history browser outside the Apify platform; storage directories are plain files you can inspect.

## API ergonomics
Minimal JS example, verbatim (https://crawlee.dev/js/docs/introduction/first-crawler):
```javascript
import { CheerioCrawler } from 'crawlee';

const crawler = new CheerioCrawler({
    async requestHandler({ $, request }) {
        const title = $('title').text();
        console.log(`The title of "${request.url}" is: ${title}.`);
    }
})

await crawler.run(['https://crawlee.dev']);
```
Router pattern, verbatim (https://crawlee.dev/js/docs/introduction/refactoring, `routes.mjs` excerpt):
```javascript
import { createPlaywrightRouter, Dataset } from 'crawlee';

export const router = createPlaywrightRouter();

router.addHandler('CATEGORY', async ({ page, enqueueLinks, request, log }) => {
    log.debug(`Enqueueing pagination for: ${request.url}`);

    await page.waitForSelector('.product-item > a');
    await enqueueLinks({
        selector: '.product-item > a',
        label: 'DETAIL',
    });

    const nextButton = await page.$('a.pagination__next');
    if (nextButton) {
        await enqueueLinks({
            selector: 'a.pagination__next',
            label: 'CATEGORY',
        });
    }
});

router.addDefaultHandler(async ({ request, page, enqueueLinks, log }) => {
    log.debug(`Enqueueing categories from page: ${request.url}`);

    await page.waitForSelector('.collection-block-item');
    await enqueueLinks({
        selector: '.collection-block-item',
        label: 'CATEGORY',
    });
});
```
Minimal Python example, verbatim (https://crawlee.dev/python/docs/quick-start):
```python
import asyncio

from crawlee.crawlers import BeautifulSoupCrawler, BeautifulSoupCrawlingContext

async def main() -> None:
    crawler = BeautifulSoupCrawler(max_requests_per_crawl=10)

    @crawler.router.default_handler
    async def request_handler(context: BeautifulSoupCrawlingContext) -> None:
        context.log.info(f'Processing {context.request.url} ...')
        data = {
            'url': context.request.url,
            'title': context.soup.title.string if context.soup.title else None,
        }
        await context.push_data(data)
        await context.enqueue_links()

    await crawler.run(['https://crawlee.dev'])

if __name__ == '__main__':
    asyncio.run(main())
```
- Delightful: no explicit ids (queue + `uniqueKey` derived from URL); one context object per handler carrying `request`, `$`/`page`/`soup`, `enqueueLinks`, `pushData`, `log`, `session`; labels turn a crawl into a small state machine without a graph DSL; retries/failed handlers are two options; autoscaling means you rarely set concurrency; swapping Cheerio for Playwright is a class name; storages are directories you can `ls`.
- Painful / footguns: `purgeOnStart` defaults to wiping the default queue/dataset/KVS, so accidental restarts lose progress unless you name storages or set the flag (Python example page exists precisely for this); Dataset is append-only, so re-run handlers duplicate rows; retry has no backoff and no `Retry-After` handling; `useState()` without an id is shared between crawlers (warning logged); JS docs describe `entries.json` while 3.x source persists per-entry files and 4.x moves the format into a native crate — the on-disk contract is not stable across majors; Python `run()` purges the queue on a second successful `run()` of the same crawler instance unless told otherwise; lock semantics differ by storage client (immediate reclaim vs 300 s lease vs wall-clock lock).
- Scraping pipeline (fetch N pages per source, parse, enrich, publish snapshot) in Crawlee: seed N requests with `label: 'PAGE'` and `userData: { sourceId }`; the PAGE handler parses and `pushData`s; enrichment is either inline in the handler or a second labelled request per item; "publish snapshot" has no hook — you do it after `run()` returns by reading the Dataset and writing your own snapshot. Plumbing you write yourself: per-source completion (count handled vs expected in `useState`), atomic snapshot/pointer, dedup of duplicated rows after crash re-runs, cooldown/`Retry-After` handling, cross-run reuse (no response cache; `uniqueKey` dedup only within a queue), and any code-version gating.

## Known limitations
- Default storages purged on start (`ConfigurationOptions.purgeOnStart`, Python `Configuration.purge_on_start`).
- No backoff/delay policy for retries documented (`BasicCrawlerOptions`, `BasicCrawler`); `maxRequestRetries` is a count only.
- Crash re-runs a handler from scratch and cannot dedupe its `pushData` (Dataset append-only, `guides/result-storage`).
- JS 4.x is still `rc` on npm (`4.0.0-rc.0`; `latest` is 3.18.1) and its storage format lives in a native Rust package "Not intended for direct use in crawlers".
- Single-process assumption unless `requestQueueAccess: 'shared'` (JS 4) or SQL/Redis clients (Python).
- No scheduler, no durable timers, no versioning.

## Relevance to Flowyard
- Borrow: `uniqueKey` semantics (normalized URL default, extended key opt-in, explicit override) as the acquisition observation key; `wasAlreadyPresent`/`wasAlreadyHandled` as the return shape for "record this item in the work set"; `label` + router as the model for a small number of typed handlers per source; `errorHandler` (before retry, can mutate the request) and `failedRequestHandler` (terminal) as the two failure hooks an activity policy should expose; the explicit 'single' vs 'shared' ownership switch with "reclaim in-progress immediately on open" for the single-owner case — this is exactly Flowyard's exclusive-owner recovery for repeat-safe work; per-request `noRetry`/`maxRetries` overrides; persisted `SessionPool` as precedent for persisting politeness state; the one-object context (`ActivityContext` should feel like `CrawlingContext`).
- Avoid: purge-by-default; re-running a handler after a crash with no record that it partially ran (Flowyard's attempt records must exist before execution); append-only result sinks as the publication model; retry counts without next-attempt times; on-disk formats that change per major without a migration path; in-memory-only domain throttles.
- Answers to open questions:
  - Q1 (unreached recorded ops): n/a; queue entries are consumed, never replayed.
  - Q2 (singleton per key): per request, yes (`uniqueKey`); per source/run, no — you name a queue and set `requestQueueAccess: 'single'` (JS 4) which asserts sole ownership but does not enforce it with a lock (unverified for the native crate).
  - Q3 (daemon vs one-shot): one-shot `run()`; `keepAlive` turns it into a daemon fed by others.
  - Q4 (inline vs queue): persisted queue pulled by an in-process pool — handlers are dispatched, not inlined in an orchestration future; there is no orchestration future.
  - Q5 (per-host cooldowns): in-memory (`sameDomainDelaySecs` by registrable domain; `maxRequestsPerMinute` global); session state persisted in KVS.
  - Q6 (checkpoint granularity): the request. Nothing finer.
  - Q7 (large outputs): files per item/record in the storage dir; SQL client stores `data` Text / `value` LargeBinary inline.
  - Q8 (code-change gate): none.
  - Q9 (retry state): attempt count persisted with the request; no next-retry time or jitter.
  - Q10 (missed runs): no scheduler.

## Sources
- https://crawlee.dev/js/docs/guides/request-storage — RequestQueue, storage dir, purge
- https://crawlee.dev/js/docs/next/guides/request-storage — v4 "next" wording on run-scoped vs named storages
- https://crawlee.dev/js/docs/guides/result-storage — Dataset/KVS layout, append-only
- https://crawlee.dev/js/docs/guides/configuration — env vars
- https://crawlee.dev/js/api/core/interface/ConfigurationOptions — purgeOnStart, persistStateIntervalMillis defaults
- https://crawlee.dev/js/api/basic-crawler/interface/BasicCrawlerOptions — maxRequestRetries, handlers, sameDomainDelaySecs
- https://crawlee.dev/js/api/core/class/AutoscaledPool and https://crawlee.dev/js/api/core/interface/AutoscaledPoolOptions — defaults
- https://crawlee.dev/js/api/core/class/SessionPool — persistence keys
- https://crawlee.dev/js/api/core/class/Request — uniqueKey rules
- https://crawlee.dev/js/api/core/class/RequestQueue — uniqueness statement, methods
- https://crawlee.dev/js/docs/guides/session-management, https://crawlee.dev/js/docs/guides/scaling-crawlers
- https://crawlee.dev/js/docs/introduction/first-crawler, https://raw.githubusercontent.com/apify/crawlee/master/docs/introduction/08-refactoring.mdx — verbatim examples
- https://raw.githubusercontent.com/apify/crawlee/master/packages/basic-crawler/src/internals/basic-crawler.ts — retry flow, defaults
- https://raw.githubusercontent.com/apify/crawlee/v3.15.0/packages/memory-storage/src/resource-clients/request-queue.ts — v3 lock/orderNo model
- https://raw.githubusercontent.com/apify/crawlee/master/packages/core/src/memory-storage/resource-clients/request-queue.ts — v4 in-memory backend
- https://raw.githubusercontent.com/apify/crawlee/master/packages/fs-storage/src/file-system-storage.ts — requestQueueAccess single/shared
- https://raw.githubusercontent.com/apify/crawlee/master/packages/core/src/storages/request_manager.ts — IRequestManager
- https://crawlee.dev/python/docs/quick-start, https://crawlee.dev/python/docs/guides/storages, https://crawlee.dev/python/docs/guides/storage-clients, https://crawlee.dev/python/docs/examples/resuming-paused-crawl
- https://crawlee.dev/python/api/class/BasicCrawler, https://crawlee.dev/python/api/class/Configuration
- https://raw.githubusercontent.com/apify/crawlee-python/master/src/crawlee/crawlers/_basic/_basic_crawler.py — run() docstring, retry flow
- https://raw.githubusercontent.com/apify/crawlee-python/master/src/crawlee/storage_clients/_file_system/_request_queue_client.py — FS layout, reclaim on open
- https://raw.githubusercontent.com/apify/crawlee-python/master/src/crawlee/storage_clients/_sql/_db_models.py and .../_sql/_request_queue_client.py — SQL schema, 300 s block
- https://registry.npmjs.org/crawlee, https://registry.npmjs.org/@crawlee/fs-storage-native, https://pypi.org/pypi/crawlee/json — versions
- Failed: https://crawlee.dev/js/api/core/interface/BasicCrawlerOptions (404; used basic-crawler path), https://raw.githubusercontent.com/apify/crawlee/master/packages/core/src/storages/request_provider.ts (404), https://raw.githubusercontent.com/apify/crawlee/master/packages/memory-storage/src/resource-clients/request-queue.ts (404 on master; used v3.15.0 tag), https://raw.githubusercontent.com/apify/crawlee/master/packages/fs-storage/src/resource-clients/request-queue.ts (adapter only; native format not fetched).
