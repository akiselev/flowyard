# Worked example: custom compiled CLI

## Goal

An application should be able to:

- link hot Rust functions and actors directly;
- use the same Flow IR/KDL and Artifactum execution semantics;
- mount all or selected framework commands;
- project functions/actor methods into domain commands;
- retain external Outboard plugins for optional implementations.

## Conceptual dependencies

```toml
[dependencies]
flow = { version = "...", features = ["runtime"] }
flow-cli = "..."
flow-cli-clap = "..."       # optional existing-CLI adapter
auction-domain = { path = "../auction-domain" }
auction-functions = { path = "../auction-functions" }
outboard = "..."
artifactum = "..."
```

## Runtime composition

```rust
async fn build_runtime() -> anyhow::Result<flow::Runtime> {
    let outboard = outboard::Registry::new("auction")?;

    let functions = flow::Registry::builder()
        .linked()                 // macro registrations in linked crates
        .outboard(outboard)       // external optional implementations
        .build()?;

    let project = flow_kdl::load("Flow.kdl")?;

    flow::Runtime::builder()
        .registry(functions)
        .artifacts(artifactum::Store::open_default()?)
        .control_store(flow::control::Sqlite::open_default()?)
        .project(project)
        .build()
}
```

## Mounting framework commands

Conceptual parser-neutral API:

```rust
let runtime = build_runtime().await?;

let workflow_commands = flow_cli::Integration::builder(runtime.clone())
    .mount_at("workflow")
    .enable_catalog()
    .enable_functions()
    .enable_actors()
    .enable_runs()
    .enable_debugging()
    .build();

let app = auctions_cli::App::builder()
    .mount(workflow_commands)
    .build();

app.run().await
```

The application might expose:

```text
auctions workflow function describe auction.parse
auctions workflow plan daily/report
auctions workflow actor status govdeals
```

## Projecting domain commands

```rust
let domain = flow_cli::Projection::builder(runtime.clone())
    .function("auction.search")
        .as_command("search")
    .function("auction.generate-report")
        .as_command("report")
    .actor_method(
        "govdeals",
        "spider.crawler",
        "refresh",
    )
        .as_command("scan")
    .build();
```

Result:

```text
auctions scan --force
auctions search --query "tektronix 2465" --limit 20
auctions report --new --within 50mi
```

Arguments/help/completion still derive from the Facet service/function contracts. The commands do not shell out to `flow`.

## Direct ordinary call

Application code may bypass dynamic dispatch entirely:

```rust
let result = auction_functions::search(
    &context,
    SearchQuery::new("tektronix 2465"),
    20,
    corpus,
).await?;
```

This is appropriate inside application code/tests. It is not automatically an Artifactum action unless invoked through the runtime/action context or explicitly wrapped.

## Dynamic linked graph call

Running the KDL Flow uses the linked registration:

```text
$ auctions workflow run daily/report

implementation choices:
  auction.parse       linked:auction-functions
  auction.prefilter   linked:auction-functions
  auction.enrich      outboard:auction-enrich-gemini (Remoc)
  auction.rank        linked:auction-functions
```

The linked functions avoid RPC serialization. `auction.enrich` remains external and persistent.

## Configurable rendering

An application can retain domain-specific output while machine output stays common:

```rust
let renderers = flow_cli::Renderers::builder()
    .human_for::<SearchResults>(auctions_cli::render_search)
    .human_for::<DealReport>(auctions_cli::render_report)
    .json_default()
    .jsonl_events_default()
    .build();
```

Custom human rendering cannot alter the semantic JSON/JSONL contracts.

## Compatibility with existing Clap CLIs

A Clap adapter can either:

- augment a `clap::Command` dynamically from Contract Shapes; or
- mount a trailing-arguments subcommand and delegate parsing to the Facet command model.

The design requirement is behavioral consistency, not a specific adapter strategy:

- generated help matches function/service descriptors;
- OS strings and paths remain lossless where possible;
- JSON argument mode remains available;
- no subprocess is required;
- selected commands can be renamed/hidden.

## Operational identity

A domain command projected from a function or actor method still emits normal structured events and attempt/actor records. `auctions scan` should be visible in:

```text
auctions workflow runs
auctions workflow logs ...
auctions workflow actor status govdeals
```

A friendly command name does not create a second execution path.
