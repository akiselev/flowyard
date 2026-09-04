# KDL frontend

## Choice

KDL v2 is the recommended default textual authoring format.

KDL's node-oriented model maps directly to actors, flows, calls, maps, inputs, and targets. It supports positional arguments, named properties, nested children, comments, slash-dash disabling, and arbitrary type annotations. The Rust `kdl` crate preserves formatting and comments during edits and provides source spans, which is unusually valuable for agent-assisted configuration changes.

KDL is a frontend. Flow IR remains canonical.

## Design constraints

The frontend MUST:

- have one preferred canonical spelling for each construct;
- preserve comments and formatting on targeted edits;
- retain exact source spans;
- produce deterministic Flow IR;
- validate values through resolved Facet/Contract Shapes;
- support machine edits without rewriting unrelated text;
- avoid loops, conditionals, templates, interpolation, and arbitrary expressions;
- provide semantic formatting and linting.

## Proposed document shape

```kdl
flow-ir 1

project "auction-watch" {
    actor "govdeals" use="spider.website@^1" {
        config {
            seed (url)"https://www.govdeals.com/"

            limits {
                max-pages 300
                timeout (duration)"20m"
            }

            refresh {
                normal (duration)"3h"
                failure-backoff (duration)"15m"
            }
        }

        expect-publication "pages" type="Collection<web.html-page@1>"
    }

    flow "daily" {
        source "pages" from="actor.govdeals.pages"

        map "parse" use="auction.parse@^1" {
            each "page" from="pages.items"
        }

        map "classify" use="auction.classify@^1" {
            each "listing" from="parse.listings"
            arg "threshold" 0.72
        }

        call "rank" use="auction.rank@^1" {
            input "listings" from="classify.accepted"
        }

        target "current" from="rank.report"
    }
}
```

## Canonical syntax rules

### Node positional argument

The first positional argument identifies the declared object:

```kdl
actor "govdeals" ...
flow "daily" ...
map "parse" ...
target "current" ...
```

### Properties

Properties carry short declaration metadata:

```kdl
map "parse" use="auction.parse@^1"
```

They should not contain large nested configuration.

### Children

Children express structured config, bindings, expected publications, and repeated entries.

```kdl
call "rank" use="auction.rank@^1" {
    input "listings" from="classify.accepted"
    arg "minimum-confidence" 0.8
}
```

### Type annotations

KDL type annotations provide unambiguous semantic scalar parsing:

```kdl
seed (url)"https://example.com"
timeout (duration)"20m"
profile (path)"profiles/govdeals.kdl"
input (artifact)"@latest"
```

The resolved Contract Shape may make annotations optional. Canonical formatting should add them where ambiguity or safety justifies it, not everywhere.

### Disabled declarations

KDL slash-dash comments are useful for reversible edits:

```kdl
/- actor "experimental-source" use="spider.website@^1" { ... }
```

Disabled content does not enter Flow IR but remains in the editable document.

## Config decoding

Actor/function config blocks are not decoded into a generic KDL tree and then manually interpreted by every integration. The frontend obtains the expected Facet/Contract Shape and constructs the typed configuration.

Example mapping:

```rust
#[derive(Facet)]
struct CrawlLimits {
    max_pages: u32,
    timeout: Duration,
}
```

```kdl
limits {
    max-pages 300
    timeout (duration)"20m"
}
```

Field naming follows one declared transformation, normally Rust snake case to KDL kebab case. Explicit aliases are contract metadata and participate in compatibility rules.

## Literal versus binding disambiguation

Bindings are always introduced by binding nodes/keywords:

```kdl
input "listing" from="parse.listing"
each "page" from="pages.items"
secret "token" ref="secret:ebay-api"
actor-ref "index" ref="actor.search-index"
```

Plain values remain literals:

```kdl
arg "threshold" 0.72
arg "mode" "fast"
```

Strings beginning with `@` are not magically artifact refs unless the expected Contract Shape is an artifact reference or an explicit `(artifact)` annotation is used.

## Profiles and composition

Large reusable configurations need composition, but the frontend should avoid general templating.

Recommended semantic capabilities:

- named typed profiles;
- explicit profile reference;
- explicit ordered overlay;
- file inclusion only at well-defined document boundaries;
- no implicit directory globbing;
- no environment-variable interpolation in KDL text.

Illustrative syntax, intentionally not finalized:

```kdl
profile "standard-crawl" type="spider.website-spec@1" {
    // typed fields
}

actor "govdeals" use="spider.website@^1" {
    config profile="standard-crawl" {
        seed (url)"https://www.govdeals.com/"
    }
}
```

The resolved effective config and layer provenance are recorded. Secrets remain references.

## Why not put programming constructs in KDL

The following should not be added:

```text
if/else
for/foreach
variables
string interpolation
user functions
general expressions
network imports
dynamic evaluation
```

These features would make static editing, diagnostics, graph queries, and agent safety worse. Projects needing generated repetition should use a Rust builder or future Starlark frontend to emit the same Flow IR.

## Editing API

Agent/human tooling should expose semantic edits:

```text
flow edit actor add ...
flow edit node set-arg daily/classify threshold 0.8
flow edit node connect daily/rank listings classify.accepted
flow edit disable actor experimental-source
flow fmt
flow check
```

The editor uses the formatting-preserving KDL syntax tree and source map. It should modify the smallest relevant span, preserve comments, and report a semantic diff.

## Diagnostics

KDL diagnostics should include:

- file and exact source span;
- declaration/node path;
- resolved function/actor contract;
- expected and actual type;
- valid field/method/port names;
- typo suggestions;
- duplicate/unknown property warnings;
- policy/capability context;
- a machine-applicable edit when safe.

Unknown fields should be errors by default for contract-bearing config. Forward-compatible extension maps must be explicit.

## Formatting

`flow fmt` should:

- use deterministic indentation and line breaks;
- preserve comments;
- retain disabled nodes;
- keep semantically ordered child lists in order;
- sort unordered properties only where it does not damage authored readability;
- never change Flow IR meaning.

A separate `--canonical` form may normalize more aggressively for generated files.

## Imports and security

Includes/imports, if supported, are resolved before IR canonicalization under an explicit project root and policy. They must not:

- escape allowed roots accidentally;
- fetch network content implicitly;
- read secrets;
- depend on current working directory ambiguously;
- produce different semantics from the same locked project.

## KDL versus alternatives

### TOML

Familiar and typed, but graph nodes and repeated nested bindings become awkward. Most TOML libraries do not offer KDL's natural node model and formatting-preserving semantic edits.

### YAML

Expressive but has implicit typing, anchors, merge semantics, and syntax edge cases that are undesirable for agent-authored operational configuration.

### RON

Pleasant for Rust-shaped values, but exposes Rust serialization aesthetics and is less natural for a language-neutral graph.

### Starlark

Excellent optional programmable frontend; excessive as the default declaration format because the graph is unknown until code evaluation.

### CUE/Nickel/Jsonnet

Powerful configuration languages, but they introduce a second sophisticated type/evaluation system beside Facet/Contract Shape. Their strengths are unnecessary for the normal explicit graph.

## Source-language round trip

Flow IR cannot round-trip all KDL comments/formatting by itself. The compiled project therefore keeps:

```text
ProjectIr          semantic representation
SourceMap          semantic element -> source span
KdlDocument        editable concrete syntax tree
```

Generated or remotely received Flow IR may have no KDL syntax tree. Tooling must distinguish “semantic IR available” from “editable KDL source available.”
