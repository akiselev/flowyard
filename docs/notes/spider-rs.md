# spider-rs (the `spider` Rust crate)

Researched: 2026-09-16. Scope: Rust. `spider` 2.53.9 (`spider/Cargo.toml` on GitHub main, pushed 2026-09-16), docs.rs `Website`/`Configuration` pages, crate README, `spider/src/website.rs`, `spider/src/configuration.rs`, `spider/src/features/disk.rs`, `examples/*.rs`. The mdBook at spider-rs.github.io returned 404 for every page tried.

## Summary
`spider` is a concurrency-first crawler library: build a `Website`, call `crawl()` (links only) or `scrape()` (keeps HTML), and optionally `subscribe()` to a `tokio::sync::broadcast` channel that streams each `Page` as it is fetched. It is optimized for throughput (global semaphore sized from CPU count, HTTP-first with headless Chrome only when a page needs JS via `crawl_smart`) and for the vendor's cloud service. It keeps almost no durable state: the visited-link set can spill to SQLite under the `disk` feature, but the frontier, budgets, delays and retry counters are in memory and a crawl is not resumable after a crash. Repository: 2,718 stars, 1 open issue, MIT, very active (pushes daily).

## Deployment and process model
- Library inside the caller's tokio runtime (`tokio` with `rt-multi-thread`; wasm32 single-thread). No server. Optional `spider_worker` (feature `decentralized`) moves fetching to a separate process over HTTP (`SPIDER_WORKER=http://127.0.0.1:3030`); optional `spider_cli`, `spider_mcp`, Node/Python bindings.
- What must run: only the process holding the `Website`. Chrome features spawn/attach a browser (`CHROME_URL` for remote). `spider_cloud` feature routes through spider.cloud.
- Concurrency is per process: `SEM_SHARED: Arc<Semaphore>` sized by `calc_limits(multiplier)` from logical/physical CPUs, overridable with `SEMAPHORE_MULTIPLIER`; `with_shared_queue(true)` shares one semaphore across `Website`s ("Use a shared semaphore to evenly handle workloads. The default is false.").

## Execution model
- Explicit in-memory structure: `links_visited: Box<ListBucket>` (dedup set), `extra_links` (buffer of discovered links between iterations), `pages: Option<Vec<Page>>` (only for `scrape*`). `crawl()` loops: take a batch of links, gate each fetch on the semaphore, apply delay, fetch, parse links, filter by budget/depth/robots/allow-lists, broadcast the page, push new links.
- crate docs (`lib.rs`), verbatim: "- [`crawl`]: start concurrently crawling a site. Can be used to send each page (including URL and HTML) to a subscriber for processing, or just to gather links." "- [`scrape`]: like `crawl`, but saves the HTML raw strings to parse after scraping is complete."
- `crawl_smart()` (feature `smart`): "This runs request as HTTP until JavaScript rendering is needed. This avoids sending multiple network request by re-using the content." `crawl_raw()` skips enhancements; `crawl_sitemap()` (feature `sitemap`); `crawl_concurrent(client, handle)` for external control.
- No determinism rules; no replay. Restart = new crawl. `persist_links()` only keeps `links_visited` across chained calls in one process ("Set the crawl status to persist between the run. Example crawling a sitemap and all links after - website.crawl_sitemap().await.persist_links().crawl().await").

## Step and operation identity
- Identity of a fetch = case-insensitive URL (`CaseInsensitiveString`) in `links_visited`; an optional content `signatures: HashSet<u64>` (with `normalize`) dedups identical HTML. `with_crawl_id(String)` tags a crawl ("This does nothing without the `control` flag enabled") and selects the SQLite file name under `disk`.
- No operation keys, no mismatch detection, no notion of "recorded but unreached".

## Persistence schema
- Default: none. `LINKS_VISITED_MEMORY_LIMIT` (env, default 15_000), `EXTRA_LINKS_MEMORY_LIMIT` (default 25_000), `PAGES_MEMORY_LIMIT` (`SPIDER_MAX_RETAINED_PAGES`, default 0 = unbounded: "keeps the historical unbounded behavior").
- `disk` feature (`dep:sqlx` sqlite; `disk_native_tls` is in the default `__basic` set, so default builds include it): `features/disk.rs` opens `SqlitePool::connect_lazy(sqlite://<path>)` where path = `$SQLITE_DATABASE_URL` or the OS temp dir, file `spider.db` or `spider_<crawl_id>.db`; tables:
  ```sql
  CREATE TABLE IF NOT EXISTS resources (id INTEGER PRIMARY KEY, url TEXT NOT NULL COLLATE NOCASE);
  CREATE INDEX IF NOT EXISTS idx_url ON resources (url COLLATE NOCASE);
  CREATE TABLE IF NOT EXISTS signatures (id INTEGER PRIMARY KEY, url INTEGER NOT NULL);
  ```
  `insert_url`/`insert_signature`, `seed()` spills the in-memory set with `INSERT OR IGNORE` when `links_visited.len() >= LINKS_VISITED_MEMORY_LIMIT`, `get_all_resources`, `clear_all()` deletes the file. `set_disk_persistance(bool)` toggles keeping the DB; `get_all_links_visited()` "Links all the links visited between memory and disk."
- HTTP cache (features `cache` via `http-cache-reqwest` + cacache on disk, `cache_mem`, `etag_cache`, `chrome_remote_cache*`): response cache keyed by the HTTP cache layer, not by spider; `with_caching(bool)`; `cache_skip_browser` returns cached content without rendering.
- Pages: `Page` has URL, bytes/html, `status_code`, optional headers (`headers` feature), links (`with_return_page_links`), screenshot paths (`chrome_screenshot` writes `./storage/` or `SCREENSHOT_DIRECTORY`), WARC output (`warc` feature in defaults). Everything is in memory or on the broadcast channel; a slow subscriber loses pages (`subscribe` doc: "If the subscription is going to block or use async methods, make sure to spawn a task to avoid losing messages.").

## Crash recovery of in-flight work
`CrawlStatus` (website.rs, verbatim variant list):
```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
pub enum CrawlStatus {
    #[default]
    Start, Idle, Active, Blocked, FirewallBlocked, ServerError, ConnectError,
    RateLimited, Empty, Invalid,
    #[cfg(feature = "control")] Shutdown,
    #[cfg(feature = "control")] Paused,
}
```
- None. If the process dies, the frontier (`extra_links`, in-flight fetches) is gone; `resources` in SQLite (if `disk`) only tells you which URLs had been seen, not which were fetched successfully or handled by the subscriber. There is no lease, heartbeat or attempt record. `CrawlStatus` (`Start, Idle, Active, Blocked, FirewallBlocked, ServerError, ConnectError, RateLimited, Empty, Invalid`, plus `Shutdown, Paused` under `control`) is in-memory.
- `control` feature: "Enables the ability to pause, start, and shutdown crawls on demand." `stop()` "Stop all crawls for the website."; `reset_status()` "Reset the active crawl status to bypass websites that are blocked." `with_crawl_timeout(Option<Duration>)` "The max duration for the crawl" cancels via `run_with_crawl_timeout`.

## Retries, backoff, timeouts
- `with_retry(u8)`: "Set the retry limit for request. Set the value to 0 for no retries. The default is 0." Per-attempt logic in the fetch macros: while `page.needs_retry() && retry_count > 0`, sleep `status_delay.max(backoff)` where backoff is exponential (`backoff_delay(attempt - 1, 200, 15_000)` ms, capped by `BACKOFF_MAX_DURATION` 60 s, with jitter per the comments "A jittered delay precedes each attempt"), then retry with a fresh tab/client. `with_retry_strategy(...)` "Set a custom retry strategy that controls retry behavior per attempt. When set, this replaces the simple `Configuration::retry` counter." (`RetryStrategy::on_retry(&AttemptOutcome) -> directive` with backoff and profile/proxy key).
- Timeouts: `request_timeout` default `Some(Duration::from_secs(120))` in `Configuration::default()` (docs.rs summary said 15 s; source says 120), `default_http_connect_timeout`, `default_http_read_timeout`, `crawl_timeout`. `hedge` feature races a second client on slow requests.
- Nothing about retries is persisted. Proxy rotation on failure via `ClientRotator`/`proxy_strategy`.

## Flow control
Feature groups relevant to an acquisition activity (from `spider/Cargo.toml` `[features]` and the crate README list):

| Group | Features | Notes |
|---|---|---|
| defaults | `basic` = `sync, cookies, ua_generator, encoding, balance, real_browser, disk_native_tls, time, adaptive_concurrency, priority_frontier, dns_cache, rate_limit, request_coalesce, auto_throttle, etag_cache, warc` + `io_uring, tcp_fastopen, splice, numa, zero_copy` | `disk_native_tls` pulls sqlx 0.8 (SQLite) by default |
| Chrome | `chrome, chrome_headed, chrome_headless_new, chrome_cpu, chrome_stealth, chrome_screenshot, chrome_store_page, chrome_intercept, chrome_remote_cache*, adblock, smart` | via `chromey`; `CHROME_URL` for remote; `smart` = HTTP first, render on demand |
| WebDriver | `webdriver, webdriver_chrome, webdriver_firefox, webdriver_edge, webdriver_headed, webdriver_stealth` | `thirtyfour` |
| Caching | `cache, cache_mem, cache_chrome_hybrid*, etag_cache, robots_cache` | HTTP-cache middleware; disk cache via cacache |
| Control/state | `control, disk, disk_aws, cron, serde, decentralized` | pause/shutdown, SQLite link spill, cron runner, worker offload |
| Filtering | `regex, glob, sitemap, full_resources, subdomains/tld (config)` | allow/deny lists |
| AI/cloud | `openai, gemini, agent*, search_*, spider_cloud, llm_json` | out of scope for Flowyard |

- `with_delay(u64)` "Delay between request as ms." (default 0; applied with `tokio::time::sleep`); `with_limit(u32)` "Set a crawl page limit. If the value is 0 there is no limit." (stored as the `"*"` budget); `with_budget(Option<HashMap<&str, u32>>)` "Set a crawl budget per path with levels support /a/b/c or for all paths with "*"" (example: `("*", 15), ("en", 11), ("fr", 3)`; the doc comment says it requires a `budget` flag but no such feature exists in the current `[features]` table — treat as always on, unverified); `with_depth(usize)` "Set a crawl depth limit. If the value is 0 there is no limit." (default 25); `with_respect_robots_txt(bool)`; `with_concurrency_limit(Option<usize>)`; `adaptive_concurrency`, `rate_limit`, `auto_throttle` (`AutoThrottleConfig`, "Response-time based rate limiting"), `priority_frontier`, `request_coalesce` are default features; `with_whitelist_url`/`with_blacklist_url` (regex with `regex` feature, glob with `glob`), `with_subdomains`, `with_tld`, `with_external_domains`.
- Keying: per `Website` (i.e. per root domain) for delay/budget; global semaphore per process. Not persisted, not cross-process. robots.txt crawl-delay is honoured when `respect_robots_txt` (source: `robot_file_parser`; `robots_cache` feature).

## Fan-out, child workflows, map
- Fan-out is the crawl itself; the subscriber gets pages in completion order. `queue(capacity) -> broadcast::Sender<String>` lets a subscriber inject links mid-crawl (`examples/queue.rs` rewrites `/en/` paths to `/fr/` and sends them). `subscribe_guard()` returns a `ChannelGuard` you `inc()` per handled page so `crawl()` waits for subscribers before finishing (and before closing Chrome). No collect/ordering primitive; `get_links()`/`get_pages()` at the end are unordered sets/vecs. `examples/loop.rs` runs multiple `Website`s via `tokio::spawn` with a shared `Configuration`.

## Versioning and code change policy
- None. Configuration is a runtime struct; nothing persisted carries a version (the SQLite tables have no schema version).

## Schedules, timers, missed runs
- `cron` feature: `website.configuration.cron_str = "1/5 * * * * *"` (six-field, seconds first), `cron_type = CronType::Crawl | Scrape`, `run_cron(website).await` returns a runner (`runner.stop()`); README: "Defaults to empty string - Requires the `cron` feature flag". Purely in-process (`async_job`/`cron` deps); no persistence, no overlap policy, no missed-run catch-up (unverified: `async_job` behaviour not inspected). `examples/cron.rs` subscribes once and lets the runner re-crawl every 5 s.

## Cancellation and late results
- `stop()`, `crawl_timeout`, dropping the future. Chrome tabs are guarded (`TabCloseGuard`, "closed on cancellation or scope exit"); the `ChannelGuard` prevents the browser from closing before subscribers finish. Late pages after `stop()` may still be broadcast until the semaphore drains (inferred from the loop structure; not tested).

## Observability and tooling
- `get_status()`, `get_size()`, `get_links()`, `time` feature (per-page duration), `tracing` feature, `page_error_status_details`, `debug` example; `spider_cli` for shell use. No run history, no inspection of a prior crawl beyond the `resources` table.

## API ergonomics
Minimal example, verbatim (crate docs, https://docs.rs/spider/latest/spider/ via `spider/src/lib.rs`):
```rust
use spider::tokio;
use spider::website::Website;

#[tokio::main]
async fn main() {
    let mut website: Website = Website::new("https://spider.cloud");
    let mut rx2 = website.subscribe(16);

    tokio::spawn(async move {
        while let Ok(res) = rx2.recv().await {
            println!("- {}", res.get_url());
        }
    });

    website.crawl().await;
}
```
Builder example, verbatim (`spider/README.md`):
```rust
let mut website = Website::new("https://choosealicense.com");

website
   .with_respect_robots_txt(true)
   .with_subdomains(true)
   .with_tld(false)
   .with_delay(0)
   .with_request_timeout(None)
   .with_http2_prior_knowledge(false)
   .with_user_agent(Some("myapp/version".into()))
   .with_budget(Some(spider::hashbrown::HashMap::from([("*", 300), ("/licenses", 10)])))
   .with_limit(300)
   .with_caching(false)
   .with_external_domains(Some(Vec::from(["https://creativecommons.org/licenses/by/3.0/"].map( |d| d.to_string())).into_iter()))
   .with_headers(None)
   .with_blacklist_url(Some(Vec::from(["https://choosealicense.com/licenses/".into()])))
   .with_cron("1/5 * * * * *", Default::Default())
   .with_proxies(None);
```
- Delightful: three calls to a streaming crawl; budgets per path prefix; `crawl_smart` HTTP-first Chrome escalation; robots/depth/limit/delay as one-liners; one struct (`Configuration`) is `Clone` and reusable across sites; huge feature menu (stealth, screenshots, WARC, OpenAI/Gemini extraction, proxies, sitemap) behind Cargo features; retries with jittered exponential backoff by setting one `u8`.
- Painful / footguns: `broadcast` channel drops pages for slow subscribers (doc warns; capacity `0` maps to semaphore permits); `scrape()` holds all HTML in memory unless `SPIDER_MAX_RETAINED_PAGES` is set; default feature set is large (`io_uring`, `numa`, Chrome-adjacent deps) and pulls `sqlx`; builder methods on `&mut self` return `&mut Self` so `Website::new(..).with_x(..).build()` needs `.build().unwrap()` and `with_budget` takes `Option<HashMap<&str, u32>>` while `set_crawl_budget` takes `CaseInsensitiveString` keys; docs.rs/README/source disagree on defaults (request timeout 15 s vs 120 s; `budget` flag); the SQLite `disk` store lives in the temp dir by default and stores only URLs; cron/state/control are feature-gated with `"does nothing without the X flag"` comments; the mdBook is offline.
- Scraping pipeline (fetch N pages per source, parse, enrich, publish) with spider alone: build a `Website` per source with `with_limit`/`with_budget`/`with_depth`/`with_delay`, `subscribe`, spawn a consumer that parses and writes each page, `crawl().await`, then publish from what the consumer collected. Plumbing you write yourself: everything durable — page storage, progress, per-page outcomes, retries beyond HTTP-level, dedup across runs, snapshot publication, cancellation propagation into the consumer, backpressure (the consumer must keep up or lose pages).

## Known limitations
- No resumable frontier; only visited URLs can spill to SQLite (`features/disk.rs`, `website.rs` `seed`).
- Broadcast subscribers can miss pages (`Website::subscribe` docs).
- Retry/delay/budget state is in memory (`configuration.rs`, fetch macros).
- Cron is in-process only (`examples/cron.rs`).
- Documentation drift: mdBook 404, docs.rs summary vs source defaults, `budget` flag mentioned but absent from `[features]`.
- Vendor-coupled roadmap (spider.cloud, MCP, agents) — 100+ features to audit for a minimal build.

## Relevance to Flowyard
- Borrow: the shape of an acquisition activity — `Website` config as the activity input (limit, budget per path, depth, delay, robots, allow/deny lists, timeout), a page stream as the output with backpressure; per-path budgets as the vocabulary for "bounded crawl" recovery units (DESIGN.md §6: "bounded crawl, page set, or explicit source cursor"); `ChannelGuard`-style "crawl is not done until consumers acknowledged" for making the activity outcome mean "pages retained"; jittered exponential backoff constants (200 ms base, 15 s cap, 60 s max) as a sane default; `crawl_timeout` as the activity deadline.
- Avoid: making spider's semaphore/delay the only rate limiter — Flowyard's per-host cooldown must be enforced at the activity boundary (spider's is per `Website`, in memory); relying on `disk` SQLite as a checkpoint (it is a dedup spill, not a journal); letting `Page`s live in memory (stream to Artifactum as they arrive); exposing spider's `Configuration` directly as workflow input (it is not stable across versions and carries Chrome/AI/cloud fields).
- How it fits as an acquisition implementation behind a Flowyard activity: an `AcquireSite` activity (repeat-safe, idempotent by `(source, page-set spec)`) constructs `Website::new(url).with_limit(n).with_budget(..).with_delay(d).with_crawl_timeout(t).with_retry(r)`, subscribes with a guard, spawns a consumer that writes each page's bytes + status + headers to Artifactum and records `(url, artifact ref)` in a local manifest, awaits `crawl()`, and returns the manifest as the activity output. Cancellation calls `stop()` and drops the future; the activity is marked "repeat-safe" so an interrupted attempt is simply re-run (the HTTP `cache` feature or Artifactum's own reuse makes the re-run cheap). Chrome features stay inside the activity (browser lifetime is the activity's responsibility per DESIGN.md §3).
- Answers to open questions:
  - Q1, Q2, Q6, Q8: n/a (no journal, no keys, no versioning).
  - Q3 (daemon vs one-shot): library call; runs as long as `crawl()` (or the cron runner) is awaited.
  - Q4 (inline vs queue): inline in the caller's runtime; optional out-of-process fetcher via `spider_worker`.
  - Q5 (per-host cooldowns): in-memory per `Website` (`delay`, `auto_throttle`, `rate_limit`), process-global semaphore; not keyed by host across `Website`s unless `with_shared_queue`.
  - Q7 (large outputs): pages in memory / broadcast; HTML cache on disk only via `cache` feature; screenshots to `./storage/`.
  - Q9 (retry state): in-memory counter, jittered backoff, not persisted.
  - Q10 (missed runs): in-process cron, no catch-up (unverified).

## Sources
- https://raw.githubusercontent.com/spider-rs/spider/main/README.md — root README (config example, "How it works")
- https://raw.githubusercontent.com/spider-rs/spider/main/spider/README.md — crate README (builder example, feature list)
- https://raw.githubusercontent.com/spider-rs/spider/main/spider/Cargo.toml — version 2.53.9, features, deps
- https://raw.githubusercontent.com/spider-rs/spider/main/spider/src/lib.rs — crate docs and minimal examples
- https://raw.githubusercontent.com/spider-rs/spider/main/spider/src/website.rs — Website struct, CrawlStatus, builder docs, memory limits, sqlite setup, retry loop
- https://raw.githubusercontent.com/spider-rs/spider/main/spider/src/configuration.rs — defaults (delay 0, depth 25, request_timeout 120 s, retry)
- https://raw.githubusercontent.com/spider-rs/spider/main/spider/src/features/disk.rs — SQLite schema and path
- https://raw.githubusercontent.com/spider-rs/spider/main/examples/{budget,cron,queue,loop,example,scrape,configuration,depth}.rs — examples
- https://docs.rs/spider/latest/spider/website/struct.Website.html — method list
- https://docs.rs/spider/latest/spider/configuration/struct.Configuration.html — fields and signatures
- GitHub repo metadata via `gh api repos/spider-rs/spider` — stars, pushed_at, open issues
- Failed: https://spider-rs.github.io/spider/ (index, website.html, crawl.html, scrape.html, cron.html — all 404)
