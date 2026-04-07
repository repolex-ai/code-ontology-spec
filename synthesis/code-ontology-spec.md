---
title: "The repolex Code Ontology"
tags: [spec, code-ontology, rdf, draft]
lab.synthesis.synthesizedClaim: "A small layered RDF 1.2 vocabulary plus alignments to existing standards is sufficient for code knowledge graphs, composes with SCIP, and is implementable today."
lab.synthesis.synthesisStatus: "draft"
lab.synthesis.evidenceStrength: "moderate"
lab.synthesis.concludes: "[[code-ontology-spec]]"
lab.synthesis.authors: ["[[tr1p-l3x]]", "[[m4rq]]", "[[spacegoat]]"]
---

# The repolex Code Ontology

**An RDF 1.2 Vocabulary for Code Knowledge Graphs**

**Authors:** TR1P.L3X, ?M4RQ, SpaceG.O.A.T. (TRIPLEFORCE squad)
**Status:** Draft
**Version:** 0.1
**License:** CC-BY-4.0

---

## Abstract

The dominant code-analysis tools today represent code as property graphs (Joern's CPG), custom binary formats (CodeQL, Glean), or interchange protocols (SCIP, LSIF). None use RDF, despite RDF's strengths in federation, schema composition, provenance tracking, and standardized query (SPARQL). This specification proposes a layered RDF 1.2 vocabulary for representing source code, AST structure, language server semantics, and security analysis. The vocabulary aligns with existing standards (PROV-O, SKOS, FaBiO, schema.org) and composes with SCIP via a documented IRI mapping. It is implemented in [repolex](https://github.com/repolex-ai/repolex) and validated against real-world codebases.

---

## §1 Motivation

*[TR1P.L3X — to draft]*

- Why no dominant RDF code ontology exists
- Why this matters now (AI agents, code memory, federation)
- What property graphs / SCIP / CodeQL / Glean each get right and wrong
- The opportunity gap

## §2 Layered Architecture

*[TR1P.L3X — to draft]*

- ast-base: tree-sitter core (Node, Point, text, position properties)
- ast-x: language-agnostic AST extension (FunctionDefinition, ClassDefinition, Comment)
- lsp-x: language server protocol semantics (hover, definitions, references)
- security: anomaly detection layer (steg, vendored vs authored)
- lex-upper: upper ontology (Document, Information, Process, Event)
- Layering rationale: each layer has a single concern, downstream layers extend via subClassOf

## §3 Triple Terms for Provenance

*[?M4RQ — to draft]*

- RDF 1.2 triple terms vs RDF-star (precise distinction)
- Using `rdf:reifies` for call edges with confidence/extracted-by/version metadata
- Why this is better than parallel provenance graphs
- SPARQL examples

## §4 Alignments

*[TR1P.L3X — to draft]*

- PROV-O for provenance (Activity, Agent, Entity, wasGeneratedBy, wasAssociatedWith)
- SKOS for tag systems and concept hierarchies
- FaBiO for citations (when code references papers)
- schema.org SoftwareSourceCode for repo-level metadata
- DCTERMS for publication / licensing
- The repolex code ontology as a first-class member of this stack

## §5 SHACL Shapes as the Validation Contract

*[?M4RQ — to draft]*

- Constraints encoded directly in OWL (owl:Restriction + owl:oneOf)
- SHACL shapes as generated artifacts, not hand-edited
- Validation as documentation: shapes describe both what's required and what's allowed
- Roadmap to SHACL 1.2 Rules + Node Expressions

## §6 Worked Examples

*[?M4RQ — to draft]*

For each example: source code → AST → triples → SPARQL queries that exercise the model.

- Python: a class definition with methods
- Rust: an impl block with trait implementations
- TypeScript: a class with type parameters
- Security: a Unicode steganography anomaly in a string literal
- Cross-cutting: "find all functions that call X across all languages"

## §7 Comparison to Alternatives

*[SpaceG.O.A.T. — to draft]*

| Format | Substrate | Schema | Query | Federation |
|--------|-----------|--------|-------|-----------|
| Joern CPG | Property graph | None enforced | Custom CPGQL | None |
| CodeQL | Custom binary | QL types | QL | None |
| Glean | Custom binary | Angle types | Angle | None |
| SCIP | Protobuf | .proto schema | None (interchange only) | Via re-indexing |
| LSIF | JSON | LSIF schema | None (interchange only) | Via re-indexing |
| Stack Graphs | Custom | Hand-rolled | Custom | None |
| **repolex code ontology** | **RDF triples** | **OWL + SHACL** | **SPARQL** | **Native** |

## §7.5 SCIP Composability

*[?M4RQ — to draft]*

- SCIP symbol grammar as input
- Mapping SCIP symbols to repolex IRIs (one-to-one, lossless)
- Worked example: Sourcegraph SCIP dump → SPARQL query
- Why this matters: existing tooling continues to work, the graph just becomes RDF

## §8 Status and Implementation

*[TR1P.L3X — to draft]*

- Implemented in repolex (Rust + Python)
- Validated against 65+ open source repos (1.2M+ triples)
- Used by the TRIPLEFORCE squad as the substrate for git-lex kits
- Public ontology files at: [paths]
- Reference implementation: [link]

## §9 Future Work

*[TR1P.L3X — to draft]*

### SHACL 1.2 Rules and Node Expressions

SHACL 1.2 (currently FPWD, projected REC late 2026) introduces compositional value computation via Node Expressions and forward-chaining inference via Rules. These can replace several pieces of imperative extraction logic with declarative shape definitions.

**Canonical example:** date intelligence in fm: namespace properties. Today, git-lex extraction uses a heuristic — if a frontmatter key contains "date" or "time", try to parse the value as a typed literal. With Node Expressions, this becomes:

```turtle
fm:DateProperty a sh:NodeShape ;
  sh:targetSubjectsOf [
    a sh:NodeExpression ;
    sh:filter [ sh:expression [ sh:matches ".*[Dd]ate.*" ] ]
  ] ;
  sh:property [
    sh:datatype xsd:date ;
    sh:nodeKind sh:Literal
  ] .
```

The heuristic becomes inspectable, modifiable, and queryable — a declaration of intent in the ontology, not buried in extraction code.

### JSON-LD Distribution and Croissant Alignment

MLCommons Croissant has made RDF-via-JSON-LD the metadata layer for 700K+ ML datasets on Hugging Face and Kaggle. By exposing repolex outputs as JSON-LD, we open the AI/ML audience for code knowledge graphs without requiring them to learn Turtle or SPARQL syntax. JSON-LD output is being added to git-lex and repolex.

### Federation Across Repositories

git-lex's sync graph architecture (append-only named graphs, delta-only commits) is designed for cross-repository federation. A future version will support SPARQL federation across multiple git-lex repos, enabling queries like "find all functions that import any module defined in any repo I contribute to" without centralizing the data.

### SHACL 1.2 Adoption Path

Once oxigraph (or any other RDF 1.2 store) ships SHACL 1.2 Node Expressions support, the repolex extraction pipeline will progressively migrate heuristic logic to declarative shapes. The contract: if it's currently a special case in extraction, it should be a shape in the spec.

---

## Appendix A: Glossary

*[All authors]*

- **Triple term** — RDF 1.2 mechanism for referring to a triple as the object of another statement
- **`rdf:reifies`** — RDF 1.2 property linking a triple to its reification
- **Sync graph** — git-lex's append-only named graph capturing changes between commits
- **Kit** — a git-lex bundle of ontology + scaffold for a specific use case (soul, squad, lab, collab, claude-code)
- **Frontmatter graph** — the RDF representation of YAML frontmatter from a markdown file
- **Class graph** — a generated named graph holding type assertions for a class

## Appendix B: Namespaces

```
@prefix ast-base:   <https://repolex.ai/ontology/extracts/tree-sitter/tree-sitter/v0.25/core/> .
@prefix ast-x:      <https://repolex.ai/ontology/repolex/ast-extension/> .
@prefix lsp-x:      <https://repolex.ai/ontology/repolex/lsp-extension/> .
@prefix security:   <https://repolex.ai/ontology/repolex/security/> .
@prefix lex-upper:  <https://repolex.ai/ontology/lex-upper/> .
@prefix prov:       <http://www.w3.org/ns/prov#> .
@prefix skos:       <http://www.w3.org/2004/02/skos/core#> .
@prefix fabio:      <http://purl.org/spar/fabio/> .
@prefix schema:     <https://schema.org/> .
@prefix dcterms:    <http://purl.org/dc/terms/> .
```

## Appendix C: Bibliography

*[All authors — accumulated as we go]*
