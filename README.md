# The repolex Code Ontology

**An RDF 1.2 Vocabulary for Code Knowledge Graphs**

A formal specification for representing source code, AST structure, language server semantics, and security analysis as RDF triples. Authored by the [TRIPLEFORCE squad](https://github.com/repolex-ai) and built on [repolex](https://github.com/repolex-ai) and [git-lex](https://github.com/repolex-ai/git-lex).

## Status

**Draft** — actively being written. This is the canonical home for the spec.

## Why

The dominant code-analysis tools today are not RDF-based:

- **Joern** uses property graphs (CPG)
- **GitHub CodeQL** uses a custom store
- **Sourcegraph SCIP** is a Protobuf serialization for code indexing
- **GitHub code-nav** uses something custom
- **Stack Graphs** uses a custom format

There is **no dominant RDF code ontology**. This spec proposes one — aligned with RDF 1.2 triple terms, PROV-O, SKOS, FaBiO, and schema.org — that captures code structure, semantics, and provenance in a single composable vocabulary.

## What's Inside

This is a [git-lex](https://github.com/repolex-ai/git-lex) repository using the [`lab` kit](https://github.com/repolex-ai/git-lex-kit-lab). The spec is structured as a research investigation with worked examples:

- **`investigation/`** — the research question being answered
- **`hypothesis/`** — claims the spec makes
- **`survey/`** — related work and prior art
- **`experiment/`** — worked examples (Python, Rust, TypeScript, security)
- **`synthesis/`** — the spec itself (final document)
- **`deliberation/`** — design discussions and tradeoffs

## Authors

- **TR1P.L3X** — structure, alignments, motivation
- **?M4RQ** — triple terms, SHACL contract, SPARQL examples, SCIP composability
- **SpaceG.O.A.T.** — implementation, comparative positioning

With contributions from the broader TRIPLEFORCE squad.

## Contributing

This spec is being drafted in public. PRs welcome.

## Related

- [repolex](https://github.com/repolex-ai/repolex) — the parser implementation
- [git-lex](https://github.com/repolex-ai/git-lex) — the storage layer
- [git-lex-kit-lab](https://github.com/repolex-ai/git-lex-kit-lab) — the lab kit (used by this repo)

## License

CC-BY-4.0
