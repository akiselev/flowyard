# Scrapy

Researched: 2026-09-16. Scope: Python. Scrapy 2.19.0 (PyPI latest); docs.scrapy.org "latest"; source from GitHub `scrapy/scrapy` master (`core/scheduler.py`, `dupefilters.py`, `squeues.py`, `pqueues.py`, `extensions/spiderstate.py`, `core/downloader/__init__.py`, `downloadermiddlewares/retry.py`, `settings/default_settings.py`, `utils/request.py`) and `scrapy/queuelib` master.

## Summary
Scrapy is a single-process, Twisted-reactor crawling framework: a Spider yields Requests and items, a Scheduler holds the frontier, a Downloader with per-slot concurrency/delay fetches, and items flow through pipelines and feed exporters. It optimizes for polite, high-throughput crawling of one site per spider run, not for durable execution. Persistence is opt-in (`JOBDIR`) and is explicitly a pause/resume facility, not crash recovery: the docs say unclean shutdown "can lead to data corruption in the job directory". Nothing about a handler's side effects is recorded; a request is the unit of work and it is either still in the frontier or gone.

## Deployment and process model
- Library + CLI (`scrapy crawl <spider>`). One process, one Twisted reactor, non-blocking I/O ("Scrapy is written with Twisted, a popular event-driven networking framework for Python").
- Nothing else needs to run. A crawl is a one-shot process that exits when the scheduler is empty (or a `CLOSESPIDER_*` condition fires). No daemon, no worker pool, no external queue.
- Data flow (docs, numbered): Engine gets start requests from the Spider -> schedules them in the Scheduler -> asks Scheduler for next requests -> sends to Downloader through downloader middlewares -> Response back through middlewares -> to Spider through spider middlewares -> Spider returns items and new Requests -> items to Item Pipeline, requests to Scheduler -> repeat until no requests remain.
- Multi-process is user-built (e.g. `scrapyd`, `scrapy-redis` scheduler); nothing in core coordinates two processes on one `JOBDIR` (no lock is created; unverified whether any component guards against two processes sharing a JOBDIR).

## Execution model
- Explicit persisted structure: the frontier (request queue) is data; spider callbacks are stateless functions of a Response. No replay, no step checkpointing. Determinism is not required of user code because nothing is re-executed from history; a resumed job simply continues popping the persisted queue.
- What `JOBDIR` persists (docs, verbatim list): "a scheduler that persists scheduled requests on disk", "a duplicates filter that persists visited requests on disk", "an extension that keeps some spider state (key/value pairs) persistent between batches".
- Docs on the directory: "The job directory will store all required data to keep the state of a single job (i.e. a spider run), so that if stopped cleanly, it can be resumed later."
- Request serialization rule (docs): "Request objects must be serializable with pickle, except for the callback and errback values ... Requests that cannot be serialized are kept in memory only: they are still sent, but they are lost when the crawl is paused." Callbacks "must be methods of the running spider" (they are stored by name via `Request.to_dict(spider=...)` / `request_from_dict`).
- `Scheduler.enqueue_request` (source): dupefilter check unless `request.dont_filter`; then `_dqpush` (disk); on `ValueError` (non-serializable) it logs once ("Unable to serialize request: %(request)s - reason: %(reason)s - no more unserializable requests will be logged (stats being collected)"), increments `scheduler/unserializable`, and falls back to the memory queue. `next_request` pops memory first, then disk. `close()` writes the disk-queue priority state (`_write_dqs_state`) and closes the dupefilter. `_dq()` logs "Resuming crawl (%(queuesize)d requests scheduled)" when it finds a non-empty disk queue.
- Spider state: `SpiderState` extension raises `NotConfigured` without `JOBDIR`; on `spider_opened` it loads `<JOBDIR>/spider.state` with `pickle.load`, on `spider_closed` it `pickle.dump(spider.state, f, protocol=4)`. Docs: "Upon spider closure, the contents of its state attribute are serialized into a file named spider.state in the JOBDIR folder"; "if a previously-generated spider.state file exists in the JOBDIR folder, it is loaded into the state attribute". It is written only at close, never incrementally.

## Step and operation identity
- Identity = request fingerprint. `scrapy.utils.request.fingerprint()`: SHA1 over request method, canonicalized URL, and body. Docs example: `http://www.example.com/query?id=111&cat=222` and `http://www.example.com/query?cat=222&id=111` produce the same fingerprint. "Request headers are ignored by default" (cookies often carry session ids); fragments are dropped because "servers usually ignore fragments in urls when handling requests". Options: `include_headers`, `keep_fragments`; `request.meta["verbatim_url"]` skips canonicalization. `REQUEST_FINGERPRINTER_CLASS` (default `scrapy.utils.request.RequestFingerprinter`) swaps the implementation; `REQUEST_FINGERPRINTER_IMPLEMENTATION` selects legacy vs current algorithm.
- `RFPDupeFilter` (source): `set[bytes]` of fingerprints; with a JOBDIR it opens `<JOBDIR>/requests.seen` in `a+b`, loads existing entries, and on each new fingerprint writes `len(fp).to_bytes(_SIZE_BYTES, "big") + fp`. No `flush()`/`fsync` per write in the source; the file is closed in `close()`. Log: "Filtered duplicate request: %(request)s - no more duplicates will be shown (see DUPEFILTER_DEBUG to show all duplicates)"; stat `dupefilter/filtered`.
- `dont_filter=True` bypasses the dupefilter entirely (retries set it; so do redirects internally). There is no "same fingerprint, different input" check: the fingerprint is the input.
- Mismatch/unreached behavior: not applicable. A queued request whose callback method no longer exists on the spider will fail at deserialization time on resume (inferred from `request_from_dict` resolving callbacks by name; not exercised).

## Persistence schema
JOBDIR layout (from the source files cited below):
```text
<JOBDIR>/
  requests.queue/           # Scheduler._dqdir; ScrapyPriorityQueue creates one disk queue per priority
    -1/  0/  ...            # negated priority per docs; PickleLifoDiskQueue = single file, FIFO = q00000.. + info.json
    0s/                     # start requests (FIFO) get an "s" suffix
    active.json             # priorities list written by Scheduler._write_dqs_state on clean close (name per source: _write_dqs_state)
  requests.seen             # RFPDupeFilter, length-prefixed SHA1 fingerprints, opened "a+b"
  spider.state              # pickle protocol 4 of spider.state, written only in spider_closed
```
Settings that shape persistence and politeness (defaults from `default_settings.py` on master):

| Setting | Default | Effect |
|---|---|---|
| `JOBDIR` | `None` | enables disk queue, `requests.seen`, `spider.state` |
| `SCHEDULER_DISK_QUEUE` | `scrapy.squeues.PickleLifoDiskQueue` | per-priority disk queue class |
| `SCHEDULER_MEMORY_QUEUE` | `scrapy.squeues.LifoMemoryQueue` | fallback for unserializable requests |
| `SCHEDULER_PRIORITY_QUEUE` | `scrapy.pqueues.DownloaderAwarePriorityQueue` | prefers least-loaded slot |
| `DUPEFILTER_CLASS` | `scrapy.dupefilters.RFPDupeFilter` | fingerprint dedup |
| `REQUEST_FINGERPRINTER_CLASS` | `scrapy.utils.request.RequestFingerprinter` | SHA1(method, canonical URL, body) |
| `CONCURRENT_REQUESTS` / `_PER_DOMAIN` | 16 / 8 | global / per-slot concurrency |
| `DOWNLOAD_DELAY` / `DOWNLOAD_DELAY_JITTER` / `RANDOMIZE_DOWNLOAD_DELAY` | 0 / 0.5 / True | per-slot delay with jitter |
| `DOWNLOAD_SLOTS` | `{}` | per-slot `delay`/`concurrency`/`jitter` overrides |
| `RETRY_TIMES` / `RETRY_HTTP_CODES` / `RETRY_PRIORITY_ADJUST` | 2 / `[500,502,503,504,522,524,408,429]` / -1 | retry policy |
| `AUTOTHROTTLE_ENABLED` / `_START_DELAY` / `_MAX_DELAY` / `_TARGET_CONCURRENCY` | False / 5.0 / 60.0 / 1.0 | adaptive per-slot delay |
| `HTTPCACHE_ENABLED` / `_POLICY` / `_STORAGE` / `_EXPIRATION_SECS` | False / DummyPolicy / FilesystemCacheStorage / 0 | response cache |
| `DOWNLOAD_TIMEOUT` | 180 | per-request timeout |
| `DEPTH_LIMIT` / `DEPTH_PRIORITY` | 0 / 0 | depth cap and priority adjust |
| `CLOSESPIDER_TIMEOUT` / `_ITEMCOUNT` / `_PAGECOUNT` / `_ERRORCOUNT` | 0 | bounded-crawl stop conditions |

- Files, not a database. Layout under `JOBDIR`:
  - `requests.queue/` — created by `Scheduler._dqdir`; inside, `ScrapyPriorityQueue` creates one downstream queue per priority at `<key>/<priority>` (start requests get an `s` suffix: `f"{self.key}/{key}s"`), and `close()` returns the list of active priorities which the scheduler serialises to JSON (`_write_dqs_state`). The docs note directories are named after the negated priority (e.g. `-1`).
  - `requests.seen` — dupefilter fingerprints (binary, length-prefixed).
  - `spider.state` — pickle of `spider.state` dict.
- Queue implementations (`scrapy/squeues.py`): `PickleFifoDiskQueue`/`PickleLifoDiskQueue` (pickle protocol 4; `PicklingError`/`AttributeError`/`TypeError` are converted to `ValueError`), `MarshalFifoDiskQueue`/`MarshalLifoDiskQueue`, `FifoMemoryQueue`/`LifoMemoryQueue`. Defaults: `SCHEDULER_DISK_QUEUE = "scrapy.squeues.PickleLifoDiskQueue"`, `SCHEDULER_MEMORY_QUEUE = "scrapy.squeues.LifoMemoryQueue"`, `SCHEDULER_START_DISK_QUEUE = "scrapy.squeues.PickleFifoDiskQueue"`, `SCHEDULER_START_MEMORY_QUEUE = "scrapy.squeues.FifoMemoryQueue"`, `SCHEDULER_PRIORITY_QUEUE = "scrapy.pqueues.DownloaderAwarePriorityQueue"`.
- queuelib on-disk format: `FifoDiskQueue` = chunk files `q%05d` plus `info.json` (`chunksize`, `size`, `tail`, `head`), 4-byte big-endian size header per item (`">L"`); `LifoDiskQueue` = single file with size header, `pop` truncates. Neither calls `fsync` (checked in `queuelib/queue.py`). `close()` on an empty queue deletes its files.
- Request records carry `meta` (including `retry_times`, `depth`, `download_slot`), `priority`, `dont_filter`, `cb_kwargs`, callback/errback names, headers, body, cookies. Responses are not persisted by the scheduler; `HttpCacheMiddleware` (below) is the only response store.
- Large outputs: items go to pipelines/feeds; the framework holds no item history.

## Crash recovery of in-flight work
- Docs (Persistence gotchas), verbatim: "Job pausing and resuming is only supported when the spider is paused by stopping it cleanly. Forced, sudden or otherwise unclean shutdown can lead to data corruption in the job directory, which may prevent the spider from resuming correctly."
- Also: "A job must be resumed with the same Scrapy version that paused it; after upgrading or downgrading Scrapy, start a new job with a new job directory." And: cookies may expire if resuming is delayed.
- What actually happens at a kill (from the source, not from docs): requests popped from the disk queue and in the downloader are gone (the LIFO file was truncated on pop); memory-queue requests are gone; fingerprints appended to `requests.seen` may or may not have reached the OS buffer (no flush); `spider.state` is whatever the last clean close wrote; `info.json` for FIFO queues may be stale. There is no lease, heartbeat, or in-progress marker for a request being downloaded.
- Nothing lets the user declare a request idempotent/retryable/manual for the crash case; the only knobs are for HTTP-level retries.
- Mid-run pause is signal-based: one SIGINT/SIGTERM triggers graceful stop (finishes in-flight downloads, runs `close`); a second forces shutdown (documented in the `scrapy crawl` docs; standard Scrapy behavior).

## Retries, backoff, timeouts
- `RetryMiddleware` defaults (`default_settings.py`): `RETRY_ENABLED = True`; `RETRY_TIMES = 2  # initial response + 2 retries = 3 requests`; `RETRY_HTTP_CODES = [500, 502, 503, 504, 522, 524, 408, 429]`; `RETRY_PRIORITY_ADJUST = -1`; `RETRY_EXCEPTIONS` = connection/DNS/timeout/`ResponseDataLossError`/`OSError`/`TunnelError` list; `RETRY_GIVE_UP_LOG_LEVEL = "ERROR"`.
- `get_retry_request(request, *, spider, reason, max_retry_times, priority_adjust, ...)` (source): reads `request.meta["retry_times"]` (default 0) and increments it; `max_retry_times` from `request.meta` else `RETRY_TIMES`; returns a copy with `meta["retry_times"]` set, `priority += priority_adjust`, `dont_filter = True`; stats `retry/count`, `retry/reason_count/<reason>`, `retry/max_reached`; log "Retrying %(request)s (failed %(retry_times)d times): %(reason)s" / "Gave up retrying %(request)s (failed %(retry_times)d times): %(reason)s". Per-request overrides: `meta["max_retry_times"]`, `meta["dont_retry"]`, `meta["priority_adjust"]`.
- Persistence of retry state: the attempt count lives in `request.meta["retry_times"]`, so it is serialised with the request when the retry copy is re-enqueued to the disk queue. There is no next-attempt time and no backoff: a retry is re-scheduled immediately at lower priority; delay comes only from the slot's `DOWNLOAD_DELAY`/AutoThrottle. 429 is just another retryable code; `Retry-After` is not honoured by the default middleware.
- Failure classification: by HTTP status list and exception class list; user code can call `get_retry_request` from a callback for custom conditions.
- Timeouts: `DOWNLOAD_TIMEOUT = 180`, per-request `meta["download_timeout"]`. Spider-level: `CLOSESPIDER_TIMEOUT`, `CLOSESPIDER_TIMEOUT_NO_ITEM`, `CLOSESPIDER_ITEMCOUNT`, `CLOSESPIDER_PAGECOUNT`, `CLOSESPIDER_PAGECOUNT_NO_ITEM`, `CLOSESPIDER_ERRORCOUNT` (all default 0). Docs: "When a certain closing condition is met, requests which are currently in the downloader queue (up to CONCURRENT_REQUESTS requests) are still processed."

## Flow control
- Global: `CONCURRENT_REQUESTS = 16` ("Use 0 for no limit"). Per slot: `CONCURRENT_REQUESTS_PER_DOMAIN = 8`, `DOWNLOAD_DELAY = 0` ("Minimum seconds to wait between 2 consecutive requests to the same domain"), `RANDOMIZE_DOWNLOAD_DELAY = True`, `DOWNLOAD_DELAY_JITTER = 0.5`. `DOWNLOAD_SLOTS = {}` lets you set `delay`, `concurrency`, `jitter` per slot name. `CONCURRENT_REQUESTS_PER_IP` is reported as deprecated in master's `default_settings.py` (a warning points to `CONCURRENT_REQUESTS_PER_DOMAIN`; `DownloaderAwarePriorityQueue` raises `ValueError` "does not support CONCURRENT_REQUESTS_PER_IP").
- Slot keying (`Downloader.get_slot_key`): `request.meta["download_slot"]` if set, else `urlparse_cached(request).hostname` (else resolved IP when IP concurrency is on). `Slot` holds `concurrency`, `delay`, `jitter`, `active`, `queue`, `transferring`, `lastseen`; `free_transfer_slots() = concurrency - len(transferring)`; `download_delay() = max(0.0, delay * (1 + random.uniform(-jitter, jitter)))`; `_process_queue` computes `penalty = delay - now + lastseen` and `call_later`s when positive. Idle slots are garbage-collected every 60 s. All of this is in-memory, per process; nothing about slots or delays is persisted, so a resumed job starts every host at full speed.
- AutoThrottle (extension, off by default): per-slot adaptive delay. Docs algorithm: start at `AUTOTHROTTLE_START_DELAY` (5.0); target delay = latency / `AUTOTHROTTLE_TARGET_CONCURRENCY` (1.0); next delay = average of previous and target; non-200 responses may only increase the delay; clamp between `DOWNLOAD_DELAY` and `AUTOTHROTTLE_MAX_DELAY` (60.0). `AUTOTHROTTLE_DEBUG` prints per-response stats. State is per slot in memory.
- Priority: `Request.priority` (int, higher first); `ScrapyPriorityQueue` one queue per priority; `DownloaderAwarePriorityQueue` also prefers "Domains (slots) with the least amount of active downloads". `DEPTH_PRIORITY`/`DEPTH_LIMIT` adjust by depth.
- Backpressure: `Downloader.needs_backout()` when `0 < total_concurrency <= len(active)`.

## Fan-out, child workflows, map
- None as a primitive. Fan-out is "yield N Requests"; join is user code (e.g. carry a counter in `cb_kwargs`/`meta`, or `spider.state`). The work set is never recorded up front (start requests are lazily iterated). Completion order is whatever the downloader produces; no per-item outcome collection.
- `CLOSESPIDER_*` are the only run-level completion policies.

## Versioning and code change policy
- Only rule stated: same Scrapy version to resume (quoted above). Spider code changes are not checked; a queued request referencing a renamed callback breaks at resume (inferred, see identity section). No migration tooling for a JOBDIR.

## Schedules, timers, missed runs
- None in core. `DOWNLOAD_DELAY` is the only timer concept. Scheduling crawls is external (`scrapyd`, cron, Zyte). No missed-run policy exists.

## Cancellation and late results
- Graceful stop on first signal: scheduler stops handing out requests, in-flight downloads finish, `close_spider` runs, JOBDIR state is written. Second signal: immediate stop (state loss). `CloseSpider` exception from a callback/pipeline closes with a reason. Late results after a forced stop simply never happen; there is no attempt identity to reject them.

## Observability and tooling
- Stats collector (`scheduler/enqueued/disk|memory`, `scheduler/dequeued/*`, `scheduler/unserializable`, `dupefilter/filtered`, `retry/*`, `downloader/*`, `httpcache/*`), dumped at close; `PeriodicLog` extension logs deltas/stats as JSON on `LOGSTATS_INTERVAL`. Telnet console, `scrapy shell`, `scrapy parse`. No CLI to inspect a JOBDIR's queue (files are pickled chunks), no per-request history, no "explain why this was cached".

## API ergonomics
Minimal example (docs Overview, https://docs.scrapy.org/en/latest/intro/overview.html):
```python
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        "https://quotes.toscrape.com/tag/humor/",
    ]

    def parse(self, response):
        for quote in response.css("div.quote"):
            yield {
                "author": quote.xpath("span/small/text()").get(),
                "text": quote.css("span.text::text").get(),
            }

        next_page = response.css('li.next a::attr("href")').get()
        if next_page is not None:
            yield response.follow(next_page, self.parse)
```
Run: `scrapy runspider quotes_spider.py -o quotes.jsonl`. With persistence: `scrapy crawl somespider -s JOBDIR=crawls/somespider-1`.

- Delightful: generators as the whole control surface (yield item or yield request); `response.follow`; settings-driven politeness (`DOWNLOAD_DELAY`, `CONCURRENT_REQUESTS_PER_DOMAIN`, AutoThrottle) with zero code; `HttpCacheMiddleware` dummy policy for free replays during development ("Every request and its corresponding response are cached. When the same request is seen again, the response is returned without transferring anything from the Internet"); `FEEDS` with `%(name)s`/`%(time)s`/`%(batch_id)d` URIs; pipelines as a numbered chain.
- Painful / footguns: JOBDIR is not crash-safe (quoted warning); `spider.state` written only at close; lambdas/closures as callbacks silently demote requests to memory-only ("they are lost when the crawl is paused"); retries have no delay and ignore `Retry-After`; per-host delay state evaporates on restart; file feeds append by default (`overwrite: False` for `file://`) so a resumed run appends duplicates for re-parsed pages, while FTP/S3/GCS "overwrites" and uses delayed delivery ("Scrapy writes items into a temporary local file, and only once all the file contents have been written (i.e. at the end of the crawl) is that file uploaded") so a crash loses the whole feed unless `batch_item_count` is set; `process_item` forgetting to return the item yields `None` downstream; dupefilter is fingerprint-only so re-fetching a changed page needs `dont_filter=True` and then you get no dedup at all.
- Scraping pipeline (fetch N pages per source, parse, enrich, publish snapshot) in Scrapy: one spider per source with `start_urls`; `parse` yields items and follow-ups; enrichment as a downloader middleware or an `async` pipeline (pipelines may await); publish as a `close_spider` step or a feed export. Plumbing you write yourself: any per-source completion/coverage accounting (`spider.state` counters), atomic snapshot publication (feeds are streams, not snapshots), idempotent item sinks, cross-run reuse (the HTTP cache is keyed only by fingerprint and expires by `HTTPCACHE_EXPIRATION_SECS` = 0 meaning never), and crash safety (checkpoint your own progress in a pipeline, or rely on the HTTP cache to make re-crawls cheap).

What a user must add to get crash-safe resumption (synthesis from the above):
1. `JOBDIR` plus only pickle-able requests with spider-method callbacks.
2. Enable `HttpCacheMiddleware` (`HTTPCACHE_ENABLED = True`, dummy policy) so that after an unclean stop, re-crawling from scratch is cheap; treat JOBDIR as advisory and delete it if the process died.
3. Idempotent item sink (upsert by an item key), because pages between the last `requests.seen` flush and the kill will be re-parsed.
4. Own progress record (pipeline writing to SQLite/DB), since `spider.state` is only flushed at clean close.
5. Own retry cooldowns for 429 (`Retry-After`) if the site demands them.

## Known limitations
- Persistence gotchas (docs `topics/jobs.html`): unclean shutdown corruption; same-version requirement; cookie expiry; non-serializable requests lost.
- No fsync in queuelib disk queues or the dupefilter file (source inspection).
- Per-slot throttling and AutoThrottle state are in-memory only (source; docs say nothing about persisting them).
- Retry policy has no backoff/`Retry-After` handling (`retry.py`).
- `DownloaderAwarePriorityQueue` "does not support CONCURRENT_REQUESTS_PER_IP" (`pqueues.py`).
- Delayed file delivery for remote feed backends loses everything on crash unless batched (`topics/feed-exports.html`).

## Relevance to Flowyard
- Borrow: request fingerprint as an explicit, documented identity function (method + canonical URL + body, headers opt-in) — a good default for Flowyard's acquisition-observation keys; per-host slots keyed by `download_slot`-style override with hostname default; `DOWNLOAD_SLOTS`-style per-host config (`delay`, `concurrency`, `jitter`); `RETRY_HTTP_CODES` as the default retryable set; AutoThrottle's latency/target-concurrency rule as an optional adaptive cooldown; `CLOSESPIDER_*` style bounded-crawl stop conditions as the natural "bounded page set" unit for spider activities; `dont_filter` as the explicit "this is a new observation of the same URL" escape hatch; stats keys as a vocabulary for `workflow inspect`.
- Avoid: pause/resume state that is only consistent on clean close; buffered writes without fsync on state files; retry state living inside the payload rather than in the store; in-memory-only cooldowns; append-only file feeds masquerading as snapshots; deriving "done" from an empty queue when in-flight items were dropped at kill time.
- Answers to open questions:
  - Q1 (unreached recorded ops): n/a — no replay; stale queued requests just get executed or fail to deserialize.
  - Q2 (singleton per key): none; JOBDIR is the run identity and nothing locks it (unverified: no lock found in scheduler/dupefilter/spiderstate source).
  - Q3 (daemon vs one-shot): one-shot process per crawl; exits when scheduler empty or `CLOSESPIDER_*` fires.
  - Q4 (inline vs queue): inline — the persisted frontier is consumed by the same reactor loop that runs callbacks; no separate worker pool.
  - Q5 (per-host cooldowns): in-memory `Slot` per hostname (or `download_slot` meta), per process, lost on restart; AutoThrottle too.
  - Q6 (checkpoint granularity): the request is the only checkpoint unit; parsing/enrichment are never checkpointed.
  - Q7 (large outputs): responses cached by fingerprint in `HTTPCACHE_DIR` directories (separate files per request: `request_body`, `request_headers`, `response_body`, `response_headers`, `meta`, `pickled_meta`); items streamed to feeds; nothing inlined in a DB.
  - Q8 (code-change gate): "same Scrapy version" only; user code unchecked.
  - Q9 (retry state): attempt count persisted only as `meta["retry_times"]` inside the queued request copy; no next-retry time, no jitter, immediate re-enqueue at lower priority.
  - Q10 (missed runs): no scheduler in core.

## Sources
- https://docs.scrapy.org/en/latest/topics/jobs.html — Jobs: pausing and resuming crawls (JOBDIR, gotchas, verbatim warning)
- https://docs.scrapy.org/en/latest/topics/scheduler.html — Scheduler API and priority queues
- https://docs.scrapy.org/en/latest/topics/settings.html — settings reference
- https://docs.scrapy.org/en/latest/topics/autothrottle.html — AutoThrottle algorithm and settings
- https://docs.scrapy.org/en/latest/topics/downloader-middleware.html — RetryMiddleware, HttpCacheMiddleware policies/storages
- https://docs.scrapy.org/en/latest/topics/request-response.html — Request params, meta keys, fingerprints
- https://docs.scrapy.org/en/latest/topics/item-pipeline.html — pipelines
- https://docs.scrapy.org/en/latest/topics/feed-exports.html — FEEDS, delayed delivery, batching
- https://docs.scrapy.org/en/latest/topics/architecture.html — data flow, Twisted
- https://docs.scrapy.org/en/latest/topics/extensions.html — CloseSpider, PeriodicLog, SpiderState
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/core/scheduler.py — Scheduler source
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/dupefilters.py — RFPDupeFilter source
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/extensions/spiderstate.py — SpiderState source
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/squeues.py — queue wrappers and serialization
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/pqueues.py — priority queues
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/core/downloader/__init__.py — Slot/Downloader source
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/downloadermiddlewares/retry.py — retry source
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/settings/default_settings.py — default values
- https://raw.githubusercontent.com/scrapy/scrapy/master/scrapy/utils/request.py — fingerprint implementation
- https://raw.githubusercontent.com/scrapy/queuelib/master/queuelib/queue.py — disk queue file format
- https://pypi.org/pypi/Scrapy/json — version 2.19.0
- https://docs.scrapy.org/en/latest/intro/overview.html — QuotesSpider example and `scrapy runspider quotes_spider.py -o quotes.jsonl` (verified verbatim)
