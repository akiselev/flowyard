# CLI, REPL, TUI, and agent interface

## Principle

The interaction surface is a structured API with multiple renderers. It is not a TUI that later receives a machine mode.

Codex exposes a non-interactive `exec` mode with stdout/JSONL, and Claude Code exposes print/headless modes with JSON and streaming JSON. Flow should fit these execution environments directly rather than requiring terminal emulation.

## Stock and embedded CLI

The stock `flow` executable is a thin use of an embeddable library:

```rust
let runtime = Runtime::builder()
    .registry(Registry::linked_and_outboard()?)
    .project(project)
    .build()?;

flow_cli::App::new(runtime).run().await
```

An application can mount selected command families:

```rust
let flow = flow_cli::Integration::builder(runtime)
    .mount_at("workflow")
    .enable_functions()
    .enable_actors()
    .enable_runs()
    .build();

app.mount(flow);
```

The exact parser adapter is open, but the command model and dispatch are parser-neutral and Facet-described. The stock implementation should use Facet/Figue capabilities where practical; a Clap adapter is valuable for existing applications.

## Domain command projection

A custom CLI may expose selected functions as first-class domain commands:

```rust
flow_cli::commands()
    .function("auction.search")
        .as_command("search")
    .function("auction.report")
        .as_command("report")
    .actor_method("govdeals", "spider.crawler", "refresh")
        .as_command("scan");
```

This avoids forcing end users to type framework vocabulary while preserving generated argument parsing and instrumentation.

## Proposed command families

```text
flow catalog

flow function list
flow function describe <id>
flow function call <id> [args]

flow actor list
flow actor describe <instance>
flow actor status <instance>
flow actor call <instance> <service> <method> [args]
flow actor restart|stop|repair <instance>
flow actor publications <instance>

flow check
flow graph [target]
flow plan [target]
flow run [target]
flow status
flow runs
flow run show <run>

flow explain <node|action|artifact|attempt>
flow lineage <artifact>
flow logs <attempt|actor>
flow attempt replay <attempt>
flow attempt fork <attempt> [overrides]
flow attempt shell <attempt>

flow repl
flow fmt
flow edit ...
flow doctor
```

Naming can be shortened after usability testing, but the resource hierarchy should remain stable in machine mode.

## Output formats

Every introspection and call command supports:

```text
--format human
--format json
--format jsonl
```

Additional global controls:

```text
--no-color
--non-interactive
--quiet
--verbose
--request-id <id>
--output <path>
```

When stdout is not a TTY, commands should default to stable non-cursor output. They should never prompt unless explicitly allowed.

## Stdout and stderr discipline

### Human mode

- primary requested result: stdout;
- progress may use stderr or an inline renderer;
- diagnostics: stderr;
- no cursor control when non-TTY.

### JSON mode

- one final JSON document on stdout;
- no human text on stdout;
- logs/diagnostics on stderr only when requested;
- stable exit status.

### JSONL mode

- all protocol events on stdout as one JSON object per line;
- each object includes request ID, sequence, timestamp, and schema version;
- stderr is reserved for launcher-level failures before protocol initialization or optional debug logs;
- terminal prompts and ANSI are prohibited.

## Event envelope

```json
{
  "protocol": "flow.events/v1",
  "request_id": "req-123",
  "sequence": 7,
  "time": "2026-09-04T10:00:00-07:00",
  "event": "progress",
  "subject": {"attempt":"attempt-456"},
  "data": {"fraction":0.4,"message":"120/300 pages"}
}
```

Required terminal events are `finished` or `failed`. Event ordering is per request. Cross-request interleaving is allowed and disambiguated by IDs.

## Exit status classes

Exact numeric values can be finalized later, but stable classes should distinguish:

- success;
- CLI/value validation failure;
- definition/contract failure;
- implementation/actor not found or incompatible;
- policy/admission rejection;
- domain execution failure;
- transport/worker failure;
- partial/indeterminate external effect;
- cancellation.

Machine clients should use structured terminal events rather than parse messages alone.

## Function CLI generation

Facet/Contract Shape drives:

- argument names and aliases;
- required/default/optional status;
- enum choices;
- nested config;
- value parser;
- help and examples;
- artifact/ref/path handling;
- secret-reference handling;
- shell completion;
- JSON request schema.

Example:

```text
$ flow function call auction.search --help

Search the auction corpus.

Usage:
  flow function call auction.search \
    --query <QUERY> \
    [--limit <LIMIT>] \
    --corpus <ARTIFACT>

Options:
  --query <QUERY>       Search expression
  --limit <LIMIT>       Maximum results [default: 20]
  --corpus <ARTIFACT>   auction.corpus@1; accepts @ref, digest, or local import
```

## REPL

### One grammar

The REPL uses the same command model as one-shot CLI operations. It is not a second programming language.

```text
$ flow repl
flow> actor govdeals status
flow> function call auction.search --query "tektronix 2465" --corpus @latest
$1 = artifact:sha256:...
flow> show $1
flow> lineage $1
```

REPL conveniences such as `$1` are session-local aliases that lower to ordinary artifact references.

### Plain mode

```text
printf '%s\n' \
  'actor govdeals status' \
  'function call auction.search --query tektronix --corpus @latest' \
  | flow repl --plain
```

Plain mode:

- accepts one command per line;
- emits bounded line-oriented output;
- has no prompt on non-TTY stdin;
- does not require terminal control sequences.

### JSONL mode

```json
{"request_id":"1","command":"actor.status","args":{"actor":"govdeals"}}
{"request_id":"2","command":"function.call","args":{"function":"auction.search","query":"tektronix","corpus":"@latest"}}
```

Responses are normal event envelopes. Multiple requests may be in flight if the caller enables concurrency.

This mode is the preferred persistent interface for coding agents that want to avoid process startup on every call.

## TUI

A richer terminal UI may provide:

- completions and contextual schema help;
- searchable catalog;
- tables/tree views;
- live progress;
- graph navigation;
- log panes;
- artifact previews.

Requirements:

- use an inline viewport by default;
- never require alternate-screen mode;
- disable itself cleanly on non-TTY streams;
- every action maps to a documented command/request;
- copyable stable IDs are always visible;
- preserve logs after completion;
- no semantic state exists only in the renderer.

A separate `flow ui` may opt into a full-screen dashboard later. `flow repl` remains automation-compatible.

## Agent discovery

An agent should be able to begin with:

```text
flow catalog --format json
```

and discover:

- functions and arguments/results;
- actor instances and services;
- current publications;
- flows and targets;
- required capabilities;
- available implementations/transports;
- examples and diagnostics schemas.

Targeted commands:

```text
flow function describe auction.classify --format json
flow actor describe govdeals --format json
flow graph daily --format json
flow plan daily --format json
```

## Bounded output and artifact spill

Large results should not flood an agent context. Commands support:

- `--limit`/pagination where semantic;
- summaries with stable artifact/log references;
- `--output <file>`;
- automatic result artifactization beyond a threshold;
- range/filter selectors;
- explicit `flow artifact show` for follow-up.

Machine output states when content was truncated and supplies continuation tokens/references.

## Prompts and confirmation

Non-interactive mode never prompts. Operations requiring confirmation fail with a structured `confirmation_required` response unless the caller supplies an explicit authorization flag/token.

Effects should support:

```text
--dry-run
--confirm <operation-digest>
```

where practical. Confirmation binds to the displayed operation identity rather than a vague global `--yes`.

## Secrets

Secrets are passed as references. Help, JSON schemas, history, logs, completions, and REPL aliases never contain secret plaintext. Reading secret values into stdout is a separately authorized operation.

## Shell and sandbox debugging

`flow attempt shell` is inherently interactive and may not suit agents. It must have structured alternatives:

```text
flow attempt materialize <id> --output <dir> --format json
flow attempt command <id> --format json
flow attempt env <id> --redacted --format json
```

The shell is convenience, not the only reproduction interface.

## Future adapters

Facet-derived descriptors can support:

- MCP tools;
- OpenAPI/HTTP endpoints;
- generated TypeScript/Python clients;
- editor/LSP completion;
- web forms.

These adapters consume the same catalog and invocation API. They should not create separate semantic contracts.
