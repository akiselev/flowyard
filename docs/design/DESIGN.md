# Flow: combined architecture design

**Status:** draft for architecture review  
**Research snapshot:** 2026-09-04  
**Scope:** design only; no implementation plan.

This is the ordered review index for the complete design. The focused chapter files are canonical so PR comments, links, and future edits have one source of truth rather than a duplicated 230 KB concatenation.

Read in order:

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

Reference material:

- [Proposed public API surface](API-SURFACE.md)
- [Glossary](GLOSSARY.md)
- [References](REFERENCES.md)
- [Worked Auctions example](examples/auctions.md)
- [Worked PartFoundry example](examples/foundry.md)
- [Custom compiled CLI example](examples/custom-cli.md)
- [Cross-language JSON plugin example](examples/cross-language-plugin.md)

The design’s central model is:

> **Facet describes values. Functions compute. Actors live. Services expose actors. Flow connects immutable results.**
