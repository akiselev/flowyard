# Spider reference design

## Why Spider is canonical

A Spider integration exercises nearly every important boundary:

- a large live runtime object that should be hydrated rather than serialized;
- rich crawl configuration;
- network/browser capabilities;
- pause/resume/shutdown control;
- streamed page events;
- keyed deduplication and snapshots;
- checkpoints and recovery;
- source-specific refresh policy;
- external website failures;
- downstream per-page parsing and classification;
- interactive manual refresh and status.

It also maps directly to Auctions, which already uses Spider, two-stage fetching, raw page caching, pure parser fixtures, and source-specific adapters.

## Contract types

### Website specification

```rust
#[derive(Facet, Contract)]
#[contract(id = "spider.website-spec", version = 1)]
pub struct WebsiteSpec {
    pub seed: Url,
    pub scope: CrawlScope,
    pub limits: CrawlLimits,
    pub http: HttpPolicy,
    pub browser: BrowserPolicy,
    pub refresh: RefreshPolicy,
    pub checkpoint: CheckpointPolicy,
}
```

### Crawl scope

```rust
#[derive(Facet, Contract)]
#[contract(id = "spider.crawl-scope", version = 1)]
pub struct CrawlScope {
    pub allowed_hosts: Vec<HostPattern>,
    pub allow_urls: Vec<UrlPattern>,
    pub deny_urls: Vec<UrlPattern>,
    pub max_depth: Option<u32>,
}
```

### Limits

```rust
#[derive(Facet, Contract)]
#[contract(id = "spider.crawl-limits", version = 1)]
pub struct CrawlLimits {
    pub max_pages: u32,
    pub max_duration: Duration,
    pub max_bytes: Option<ByteSize>,
    pub concurrency: u16,
    pub delay: Duration,
}
```

### Browser policy

```rust
pub enum BrowserMode {
    Never,
    OnDemand,
    Always,
}

pub struct BrowserPolicy {
    pub mode: BrowserMode,
    pub executable: Option<PathReference>,
    pub stealth: bool,
    pub render_timeout: Duration,
}
```

The spec describes intent and reviewed capabilities. It does not expose every internal Spider field merely because reflection makes it possible.

## Hydration

```rust
impl Hydrate for WebsiteSpec {
    type Runtime = spider::website::Website;

    async fn hydrate(
        self,
        context: &HydrateContext,
    ) -> Result<Self::Runtime, HydrateError> {
        // Validate host/network/browser policy.
        // Resolve browser executable and credentials by reference.
        // Construct Website and apply supported options.
        // Return a ready but not yet crawling runtime object.
    }
}
```

Hydration diagnostics should distinguish:

- invalid URL/scope;
- unavailable compiled Spider feature;
- missing browser binary;
- policy denial;
- invalid timeout/rate/concurrency combination;
- unsupported checkpoint behavior;
- secret or proxy resolution failure.

The actor descriptor can advertise which `WebsiteSpec` features this implementation supports.

## Page artifact

```rust
#[derive(Facet, Contract)]
#[contract(id = "web.page-observation", version = 1)]
pub struct PageObservation {
    pub canonical_url: Url,
    pub requested_url: Url,
    pub observed_at: Timestamp,
    pub status: Option<u16>,
    pub headers: HeaderSummary,
    pub body: Artifact<HtmlPage>,
    pub fetch_receipt: Artifact<FetchReceipt>,
}
```

The raw HTML bytes are an Artifactum blob with media type. URL/status/time/receipt remain provenance/observation metadata; content identity is not provenance.

## Crawler actor

```rust
#[outboard::actor(id = "spider.website", version = 1)]
pub struct SpiderCrawler {
    website: Website,
    spec: WebsiteSpec,
    state: CrawlerState,
}
```

Persistent control state may include:

```rust
pub struct CrawlerState {
    pub last_started: Option<Timestamp>,
    pub last_completed: Option<Timestamp>,
    pub last_successful_publication: Option<ArtifactId>,
    pub checkpoint: Option<ArtifactId>,
    pub consecutive_failures: u32,
    pub next_wake: Option<Timestamp>,
    pub paused: bool,
    pub active_operation: Option<OperationId>,
}
```

The live `Website`, subscriber channels, browser handles, and in-flight tasks are ephemeral and rehydrated.

## Crawler service

```rust
#[outboard::service(id = "spider.crawler", version = 1)]
pub trait CrawlerService {
    async fn status(&self) -> Result<CrawlerStatus, CrawlerError>;
    async fn refresh(&self, request: RefreshRequest)
        -> Result<CrawlOperation, CrawlerError>;
    async fn pause(&self) -> Result<(), CrawlerError>;
    async fn resume(&self) -> Result<(), CrawlerError>;
    async fn cancel(&self, operation: OperationId)
        -> Result<CancelOutcome, CrawlerError>;
}
```

`refresh` returns an operation handle/receipt rather than blocking the actor mailbox for the full crawl. The caller may subscribe to operation events or wait for completion.

## Autonomous refresh policy

The actor decides when another fetch is useful based on domain state:

```text
paused?
active operation?
last result and failure class?
configured normal interval?
source-specific active-item interval?
server rate-limit/reset time?
manual force request?
checkpoint available?
```

The generic Flow scheduler does not encode these decisions.

A minimal policy type might be:

```rust
pub struct RefreshPolicy {
    pub normal: Duration,
    pub after_failure: BackoffPolicy,
    pub minimum_spacing: Duration,
    pub startup: StartupRefresh,
}
```

Auctions may layer domain policy—for example, active auction closing windows—inside a specialized actor or policy callback rather than overgeneralizing `WebsiteSpec`.

## Crawl operation

### Admission

The actor serializes refresh admission, records operation identity, and prevents accidental overlapping crawls unless the actor type explicitly supports them.

### Execution task

The crawl runs outside the actor mailbox with:

- immutable effective `WebsiteSpec`;
- hydrated/owned crawler runtime;
- cancellation token;
- publication session handle;
- checkpoint handle;
- event sink.

### Stream processing

Spider's subscription API emits pages as they arrive. For each page:

1. derive a logical key, normally canonical URL or source-provided external ID;
2. commit raw body bytes to Artifactum;
3. commit a fetch receipt/observation record;
4. upsert the keyed item in the open `pages` publication session;
5. emit `item_published` and progress events;
6. permit downstream mapped parse/classify work.

### Completion

On successful complete snapshot:

- seal the collection;
- atomically advance `pages` publication;
- commit crawl receipt/metrics;
- update actor state/backoff/next wake;
- emit completion.

On partial source semantics, the publication contract marks the snapshot partial and downstream deletion behavior changes accordingly.

### Failure

On failure:

- preserve committed page artifacts and valid per-item derivations;
- retain checkpoint/logs/diagnostics;
- abort rather than advance an incomplete complete-snapshot publication;
- classify retry/backoff;
- update actor state;
- expose the failure through service status and events.

## Publications

Recommended publications:

```text
pages       Collection<PageObservation>
last-run    CrawlReceipt
health      CrawlerHealthSnapshot
```

`health` is a small snapshot, not a live mutable view. `status()` can provide more current operational detail.

## Downstream Flow

```kdl
flow "daily" {
    source "pages" from="actor.govdeals.pages"

    map "parse" use="auction.parse@^1" {
        each "page" from="pages.items"
    }

    map "prefilter" use="auction.prefilter@^1" {
        each "listing" from="parse.listings"
    }

    map "enrich" use="auction.enrich@^1" {
        each "listing" from="prefilter.promoted"
    }

    call "rank" use="auction.rank@^1" {
        input "listings" from="enrich.items"
    }

    target "current" from="rank.report"
}
```

A new page begins parsing before the crawl seals. `rank`, which needs the complete selected set, waits for the relevant collection seals.

## Two-stage crawling

Auctions already benefits from cheap discovery followed by local relevance/geography filtering and then detail/image fetching. The reference design should model this as multiple actors/functions rather than one giant crawler:

```text
discovery actor publishes ListingStub collection
    |
map local prefilter
    |
selection collection
    |
detail-fetch actor/service observes selected URLs
    |
map parser/enrichment
```

An actor may subscribe to a Flow target/publication and fetch selected details, but this feedback loop must be explicit and cycle-checked at the publication/run level.

## Fixture-driven parser testing

Crawler transport and parsers remain separate. A parser function consumes immutable page artifacts and can be tested against checked-in fixtures without network/browser credentials.

This preserves a key Auctions lesson: live tests tell us a site changed; fixture tests tell us our parser behavior changed.

## Interactive usage

```text
flow actor status govdeals
flow actor call govdeals spider.crawler refresh --force
flow actor publications govdeals
flow logs actor:govdeals
flow function call auction.parse --page ./fixtures/govdeals/lot.html
```

Agent mode:

```text
flow actor call govdeals spider.crawler refresh \
  --args-json '{"force":true}' \
  --format jsonl
```

## Reconfiguration

A config change produces a new actor desired generation. Reconfiguration policy is explicit:

- reject while active;
- quiesce then replace;
- apply selected live-safe fields;
- create a second actor instance.

The actor descriptor can identify live-reconfigurable fields, but the default is generation replacement. Arbitrary reflected field mutation of a live `Website` is not assumed safe.

## Capability declaration

The actor/function descriptors should expose requirements such as:

```text
network hosts
robots policy
browser/chrome feature
filesystem cache
proxy/secret refs
maximum concurrency
control support
```

The runtime can reject a KDL spec that requests a feature not compiled into the selected implementation before attempting a crawl.

## What the example proves

The canonical example is successful only if the same semantic setup works with:

- a linked Rust crawler actor in a custom Auctions CLI;
- a persistent external Rust actor over Remoc;
- a structured JSON mock/alternate implementation;
- KDL validation and editing;
- one-shot and REPL agent control;
- crash recovery without partial publication;
- per-item cache reuse across snapshots.
