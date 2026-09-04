# Worked example: Auctions

This example is illustrative and emphasizes the intended author experience rather than exact source-specific fields.

## Domain contracts

```rust
#[derive(Facet, Contract)]
#[contract(id = "auction.listing-stub", version = 1)]
pub struct ListingStub {
    pub source: SourceId,
    pub external_id: String,
    pub url: Url,
    pub title: String,
    pub observed_price: Option<Money>,
    pub location: Option<Location>,
}

#[derive(Facet, Contract)]
#[contract(id = "auction.listing", version = 2)]
pub struct Listing {
    pub stub: ListingStub,
    pub description: Option<String>,
    pub images: Vec<ImageRef>,
    pub ends_at: Option<Timestamp>,
    pub bids: Option<u32>,
    pub raw_observation: Artifact<RawListingObservation>,
}

#[derive(Facet, Contract)]
#[contract(id = "auction.promotion-decision", version = 1)]
pub struct PromotionDecision {
    pub listing: Artifact<Listing>,
    pub accepted: bool,
    pub score: f32,
    pub reasons: Vec<PromotionReason>,
}
```

## Functions

```rust
#[flow::derive(id = "auction.parse-stub", version = 1, cache = "pure")]
async fn parse_stub(
    page: Artifact<PageObservation>,
) -> Result<Collection<ListingStub>, ParseError>;

#[flow::derive(id = "auction.parse-detail", version = 2, cache = "pure")]
async fn parse_detail(
    page: Artifact<PageObservation>,
) -> Result<Listing, ParseError>;

#[flow::derive(id = "auction.prefilter", version = 1, cache = "pure")]
async fn prefilter(
    listing: Artifact<Listing>,
    policy: PrefilterPolicy,
) -> Result<PromotionDecision, ClassifyError>;

#[flow::function(id = "auction.search", version = 1)]
async fn search(
    query: String,
    #[flow(default = 20)] limit: u32,
    corpus: Artifact<AuctionCorpus>,
) -> Result<SearchResults, SearchError>;

#[flow::effect(id = "auction.send-alert", version = 1)]
async fn send_alert(
    request: AlertRequest,
) -> Result<AlertReceipt, AlertError>;
```

## Actor instances

The discovery actor owns network refresh behavior. A detail-fetch actor consumes an explicit selected-URL publication and fetches only promoted listings.

```kdl
flow-ir 1

project "auction-watch" {
    profile "common-crawl" type="spider.website-spec@1" {
        limits {
            max-pages 300
            timeout (duration)"20m"
            delay (duration)"1s"
        }

        refresh {
            normal (duration)"3h"
            failure-backoff {
                initial (duration)"5m"
                maximum (duration)"2h"
            }
        }
    }

    actor "govdeals-discovery" use="spider.website@^1" {
        config profile="common-crawl" {
            seed (url)"https://www.govdeals.com/"

            scope {
                allowed-host "www.govdeals.com"
                allow-url "/search*"
                allow-url "/asset/*"
            }
        }

        expect-publication "pages" type="Collection<web.page-observation@1>"
    }

    actor "detail-fetch" use="auction.detail-fetcher@^1" {
        config {
            input-publication "daily.selected-urls"
            concurrency 4
            minimum-spacing (duration)"1s"
        }

        expect-publication "pages" type="Collection<web.page-observation@1>"
    }

    flow "daily" {
        source "discovery-pages" from="actor.govdeals-discovery.pages"
        source "detail-pages" from="actor.detail-fetch.pages"

        map "stubs" use="auction.parse-stub@^1" {
            each "page" from="discovery-pages.items"
        }

        map "stub-score" use="auction.stub-prefilter@^1" {
            each "stub" from="stubs.items"
            arg "home" "@profile.home"
            arg "watchlists" "@profile.watchlists"
        }

        call "selected-urls" use="auction.select-detail-urls@^1" {
            input "decisions" from="stub-score.results"
        }

        publish "selected-urls" from="selected-urls.urls"

        map "listings" use="auction.parse-detail@^2" {
            each "page" from="detail-pages.items"
        }

        map "prefilter" use="auction.prefilter@^1" {
            each "listing" from="listings.items"
            arg "policy" "@profile.prefilter"
        }

        map "enrich" use="auction.enrich@^1" {
            each "decision" from="prefilter.accepted"
        }

        call "rank" use="auction.rank@^1" {
            input "items" from="enrich.items"
            arg "home" "@profile.home"
        }

        target "report" from="rank.report"
    }
}
```

`publish` in this example exposes a Flow result for an actor subscription. This is an explicit delayed feedback boundary, not an immediate graph cycle. The concrete IR needs generation semantics such as “detail-fetch consumes the latest sealed selected-URL publication and publishes a later generation.”

## User commands

```text
auctions scan
auctions deals --new
auctions search "tektronix 2465"

auctions workflow plan daily/report
auctions workflow graph daily/report
auctions workflow explain daily/prefilter:govdeals/1234
```

The domain commands can be projections:

```text
auctions scan
  -> actor service call detail/discovery refresh

auctions search
  -> linked call to auction.search
```

## Agent interaction

```text
$ auctions workflow actor status govdeals-discovery --format json
{
  "actor":"govdeals-discovery",
  "state":"ready",
  "last_success":"2026-09-04T08:00:00-07:00",
  "current_publication":"artifact:sha256:...",
  "next_wake":"2026-09-04T11:00:00-07:00"
}
```

```text
$ auctions search \
    --query "tektronix 2465" \
    --format json
```

## Explain example

```text
$ auctions workflow explain daily/prefilter:govdeals/1234

Will execute auction.prefilter@1.

listing artifact:  sha256:3e... -> sha256:8a...  changed
policy:            sha256:19...                    unchanged
implementation:    sha256:c2...                    unchanged

Reason for listing change:
  price observation changed from $75 to $110

Downstream:
  auction.enrich remains blocked because the new decision has not completed
  rank.report may change
```

## Why this remains application-owned

The generic framework does not know:

- what constitutes an auction listing;
- how to score buried treasure;
- whether a closing-soon lot is attractive;
- how geographic/pallet constraints work;
- which URLs deserve detail fetches;
- which model calls are worth paying for.

It supplies the typed execution, publication, transport, artifact, and debugging substrate.
