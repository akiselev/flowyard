# Flow design review package

**Status:** architecture draft for review  
**Research snapshot:** 2026-09-04  
**Working name:** `Flow` is provisional  
**Scope:** design only; this package deliberately contains no implementation sequence, staffing estimate, milestone plan, or task breakdown.

This package specifies a Rust-first, language-neutral data ingestion and transformation system distilled from Auctions, PartFoundry, Artifactum, Outboard, Daemonkit, Switchyard, and the proposed Outboard–Remoc work.

The design is organized around four nouns:

> **Facet describes values. Functions compute. Actors live. Services expose actors. Flow connects immutable results.**

The primary constraint is developer experience for both humans and coding agents. A new transform should normally require one typed function and a small Flow IR/KDL binding. It should not require scheduler, persistence, protocol, CLI, UI, or retry plumbing.

## Recommended review order

1. [Executive summary](00-executive-summary.md)
2. [Decision register](01-decision-register.md)
3. [Goals, scope, and principles](02-goals-scope-principles.md)
4. [Conceptual model](03-conceptual-model.md)
5. [System architecture](04-system-architecture.md)
6. [Facet contracts and hydration](05-facet-contracts-and-hydration.md)
7. [Functions](06-functions.md)
8. [Services and actors](07-services-and-actors.md)
9. [Outboard, Remoc, and JSON transports](08-outboard-remoc-and-json.md)
10. [Flow IR](09-flow-ir.md)
11. [KDL frontend](10-kdl-frontend.md)
12. [Execution, artifacts, and incrementality](11-execution-artifacts-and-incrementality.md)
13. [CLI, REPL, TUI, and agent interface](12-cli-repl-tui-and-agent-interface.md)
14. [Macros and generated code](13-macros-and-generated-code.md)
15. [Spider reference design](14-spider-reference-design.md)
16. [Repository integration boundaries](15-repository-integration-boundaries.md)
17. [Security, versioning, and compatibility](16-security-versioning-and-compatibility.md)
18. [Prior art and research conclusions](17-prior-art-and-research.md)
19. [Open questions and review checklist](18-open-questions-and-review-checklist.md)
20. [Design validation scenarios](19-design-validation-scenarios.md)

Additional material:

- [Proposed public API surface](API-SURFACE.md)
- [Glossary](GLOSSARY.md)
- [References](REFERENCES.md)
- [Worked Auctions configuration](examples/auctions.md)
- [Worked PartFoundry configuration](examples/foundry.md)
- [Custom compiled CLI example](examples/custom-cli.md)
- [Cross-language JSON plugin example](examples/cross-language-plugin.md)
- [Combined design review index](DESIGN.md)

## Normative language

`MUST`, `MUST NOT`, `SHOULD`, `SHOULD NOT`, and `MAY` indicate design requirements. Code is illustrative unless a section explicitly calls it normative.

## Core review questions

The review should concentrate on these issues:

1. Is the function/actor/service division understandable without knowing the runtime internals?
2. Does Facet remain the single Rust reflection system without accidentally becoming the wire ABI?
3. Can the same semantic API execute linked, over Remoc, and through JSON without lowest-common-denominator ergonomics?
4. Are actor publication and collection sealing semantics strong enough for incremental ingestion?
5. Is the KDL surface small enough that agents can edit it reliably without becoming a programming language?
6. Can every interactive action be driven non-interactively through stable JSON/JSONL?
7. Do Artifactum, Outboard, Daemonkit, and the workflow control plane retain non-overlapping ownership?
