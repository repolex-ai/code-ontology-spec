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

### 1.1 The Property Graph Default

When developers reach for a code-as-data tool today, they get a property graph. Joern uses Code Property Graphs (CPG). Sourcegraph SCIP is a Protobuf interchange format that targets property-graph stores. GitHub CodeQL uses a custom datalog-style query language over a custom binary format. Meta's Glean is another custom binary format. Stack Graphs has its own representation. Each tool ships its own schema, its own query language, and its own ingestion pipeline.

This is the world that the semantic web community spent twenty years arguing against, and lost.

### 1.2 What RDF Lost

The RDF stack has clear advantages for code knowledge graphs that the property-graph world has been working around piecemeal:

**Federation.** Property graphs are islands. Joern's CPG cannot be queried alongside CodeQL's database without an ETL pipeline that loses information at each step. RDF was designed for the open-world federation problem from day one — multiple graphs over a SPARQL endpoint, cross-store joins, shared vocabularies. The cost is that the RDF community built the standards before the tooling caught up; the property-graph community built tooling first and is now reinventing federation badly.

**Standardized query.** SPARQL is a W3C recommendation with multiple conforming implementations (Apache Jena, Oxigraph, GraphDB, Stardog, Virtuoso, QLever). Cypher exists but is fragmented across vendors; Neo4j's GQL push has not produced a unified successor. CodeQL, CPGQL, Angle, Glean's query language — each is bespoke. SPARQL is the only mature standardized query language for graph data, and it has been since 2008.

**Shared vocabularies.** PROV-O for provenance. SKOS for tag systems. FaBiO for citations. schema.org for metadata. DCTERMS for licensing. These are mature ontologies that any RDF graph can compose with. Property graphs have no equivalent — every project reinvents its own vocabulary, and the closest thing to a "shared schema" is convention.

**Provenance native.** RDF 1.2 (December 2025 W3C recommendation) introduced triple terms and `rdf:reifies` specifically to attach metadata to individual edges without ugly reification. This is exactly what code analysis needs: a call edge isn't just "A calls B" — it's "A calls B, extracted by tree-sitter v0.25, confidence 1.0, observed at commit abc123, by parser version 0.4." Property graphs can do this with edge properties, but the result is per-implementation; RDF 1.2 makes it standard.

The property-graph world won the tooling war. The RDF stack won the standards war. Code knowledge graphs are a domain where the standards-side should have won the tooling war too — but nobody built the tools.

### 1.3 Why Now

Three things converged in early 2026 that change the calculus:

**1. RDF 1.2 shipped.** Triple terms + `rdf:reifies` are now stable. Apache Jena 6.0.0 (February 2026) and Oxigraph 0.5.x (which removed the experimental `rdf-star` feature flag entirely) both implement RDF 1.2 by default. The version of RDF that handles provenance natively is finally available in mature implementations.

**2. AI agents need persistent code memory.** Every AI coding assistant — Cursor, Continue, Cody, Claude Code, Gemini Code Assist, Codex — has converged on the same architecture: index the codebase, store the index, query at inference time. The index is currently either vector embeddings (lossy, opaque) or a custom property graph (Sourcegraph, Glean). RDF is the right substrate for this and nobody is shipping it. The window for proposing a standard is now, before the property-graph defaults harden.

**3. SHACL 1.2 is becoming an inference engine.** SHACL split into five working drafts (Core, Rules, Node Expressions, SPARQL Extensions, UI) in early 2026. Node Expressions (FPWD 2026-01-08) introduces compositional value computation; Rules introduces forward-chaining inference. Together they turn SHACL from a validator into a declarative rule language for RDF graphs. For code analysis — where most "rules" are currently buried in imperative parser code — this is transformative.

### 1.4 Who This Spec Is For

Three audiences:

1. **Tool implementers** building AI coding assistants who want a standardized substrate for code memory instead of inventing their own format.
2. **Researchers** studying code as data who want reproducible queries and federated datasets across projects and languages.
3. **The semantic web community** who want a foothold in the developer-tools world after twenty years of being adjacent to it.

The spec is implemented in [repolex](https://github.com/repolex-ai/repolex) and [git-lex](https://github.com/repolex-ai/git-lex). It is not aspirational — it ships triples today, validates today, queries today. This document captures what we built and why, so others can adopt and federate.

## §2 Layered Architecture

*[TR1P.L3X — to draft]*

- ast-base: tree-sitter core (Node, Point, text, position properties)
- ast-x: language-agnostic AST extension (FunctionDefinition, ClassDefinition, Comment)
- lsp-x: language server protocol semantics (hover, definitions, references)
- security: anomaly detection layer (steg, vendored vs authored)
- lex-upper: upper ontology (Document, Information, Process, Event)
- Layering rationale: each layer has a single concern, downstream layers extend via subClassOf

## §3 Triple Terms for Provenance

### 3.1 The Provenance Problem in Code Graphs

A code knowledge graph is not just structure — it is *claims*. The triple `:functionA :calls :functionB` is a claim with a source, a confidence, an extraction method, and a version. A call edge derived from a tree-sitter parse is different from one derived from an LSP `textDocument/references` query, which is different from one resolved through a SCIP indexer's `is_reference` relationship. The same edge may be re-derived multiple times by different tools, with different confidence levels, at different commits.

In a property graph, this metadata is attached as edge properties — but the schema is per-implementation, the query language for traversing it is bespoke, and there is no standard way to express "this edge was derived by tool X with confidence Y." In RDF prior to 1.2, the standard answer was *reification*: invent a node representing the triple, attach metadata to that node, and accept that you have just doubled the size of your graph and broken every query that traverses by `?s :calls ?o`. Reification was correct in principle and unusable in practice.

RDF 1.2 introduced **triple terms** specifically to fix this.

### 3.2 RDF 1.2 Triple Terms (Precisely)

A triple term is an RDF term — alongside IRIs, blank nodes, and literals — that refers to a triple. Triple terms appear only in the **object position** of statements, and they are linked to their reifier via `rdf:reifies`. The Turtle annotation syntax `:s :p :o {| :p2 :o2 |}` is sugar that creates both an asserted triple and a reified triple sharing structure.

**Important precision:** RDF 1.2 is *not* a rename of RDF-star.

- **RDF-star** was a Community Group experiment that introduced `<< :s :p :o >>` as an "embedded triple" appearing anywhere in any position. It was never standardized.
- **RDF 1.2** is the W3C Working Group successor. Some findings from RDF-star informed the WG, but RDF 1.2 ships its own model: triple terms appear only in object position, are linked via `rdf:reifies`, and have *different semantics* from the embedded-triple model.

Old RDF-star data does not roundtrip cleanly to RDF 1.2 in all cases. Apache Jena's documentation explicitly warns users about this. Oxigraph 0.5 removed the experimental `rdf-star` feature flag entirely, replacing it with `rdf-12`/`sparql-12`. If a tool says "RDF-star," check whether it means the CG experiment or the new spec; the two are not interchangeable.

For this specification, we use only RDF 1.2 triple terms with `rdf:reifies`.

### 3.3 Provenance for a Call Edge

The minimal RDF 1.2 expression of a provenanced call edge:

```turtle
@prefix ast-x: <https://repolex.ai/ontology/repolex/ast-extension/> .
@prefix lsp-x: <https://repolex.ai/ontology/repolex/lsp-extension/> .
@prefix prov:  <http://www.w3.org/ns/prov#> .
@prefix rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .

# The asserted edge
<#functionA> lsp-x:calls <#functionB> .

# The reification, identified for annotation
<#call_assertion_001>
    rdf:reifies << <#functionA> lsp-x:calls <#functionB> >> ;
    prov:wasGeneratedBy <#extraction_run_42> ;
    repolex:confidence "0.95"^^xsd:decimal ;
    repolex:resolvedBy "scip-java" ;
    repolex:resolverVersion "0.10.1" ;
    repolex:observedAtCommit "abc123def456" .
```

The Turtle annotation syntax is sugar for the same shape:

```turtle
<#functionA> lsp-x:calls <#functionB>
    {| prov:wasGeneratedBy <#extraction_run_42> ;
       repolex:confidence "0.95"^^xsd:decimal ;
       repolex:resolvedBy "scip-java" ;
       repolex:resolverVersion "0.10.1" ;
       repolex:observedAtCommit "abc123def456" |} .
```

Both forms produce the same triples. The annotation syntax is preferred for ergonomics.

### 3.4 Why Not Parallel Provenance Graphs?

The pre-RDF-1.2 alternative was to maintain a parallel "provenance graph" containing one node per claim, with the original graph asserting only the bare relations. This works but has three structural problems:

1. **Joins are expensive.** Every query that wants provenance has to JOIN the structural graph to the provenance graph by some opaque key, and the join key is not part of the structural query — it's bookkeeping.
2. **Atomicity is broken.** A claim and its provenance can drift apart. There is no constraint that the provenance for a triple exists, or that it remains consistent with that triple.
3. **The query language is harmed.** SPARQL queries that want to filter by confidence have to either retrieve everything and post-filter, or write awkward subqueries that materialize the join.

Triple terms collapse the problem: the claim and its provenance share structure. A query for "all calls with confidence > 0.8 derived by SCIP" is one BGP, not two:

```sparql
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX lsp-x: <https://repolex.ai/ontology/repolex/lsp-extension/>
PREFIX repolex: <https://repolex.ai/ontology/repolex/>

SELECT ?caller ?callee ?confidence WHERE {
  ?caller lsp-x:calls ?callee .
  ?reifier rdf:reifies << ?caller lsp-x:calls ?callee >> ;
           repolex:resolvedBy "scip-java" ;
           repolex:confidence ?confidence .
  FILTER (?confidence > 0.8)
}
```

The asserted edge and its provenance are queryable in a single graph pattern. There is no separate provenance graph to maintain, no sync to keep, no join key to remember.

### 3.5 Multi-Tool Convergence on the Same Edge

The same call edge can be claimed by multiple tools. RDF 1.2 makes it natural to represent this without duplication of structural triples:

```turtle
<#functionA> lsp-x:calls <#functionB> .

<#claim_a> rdf:reifies << <#functionA> lsp-x:calls <#functionB> >> ;
    repolex:resolvedBy "tree-sitter" ;
    repolex:confidence "0.5"^^xsd:decimal .

<#claim_b> rdf:reifies << <#functionA> lsp-x:calls <#functionB> >> ;
    repolex:resolvedBy "scip-java" ;
    repolex:confidence "0.95"^^xsd:decimal .

<#claim_c> rdf:reifies << <#functionA> lsp-x:calls <#functionB> >> ;
    repolex:resolvedBy "jdtls" ;
    repolex:confidence "0.99"^^xsd:decimal .
```

Three claims, one structural edge. Convergence of multiple tools on the same conclusion is itself information — a query can ask "how many distinct tools claimed this edge?" and answer it directly:

```sparql
SELECT ?caller ?callee (COUNT(DISTINCT ?tool) AS ?agreement) WHERE {
  ?caller lsp-x:calls ?callee .
  ?claim rdf:reifies << ?caller lsp-x:calls ?callee >> ;
         repolex:resolvedBy ?tool .
}
GROUP BY ?caller ?callee
ORDER BY DESC(?agreement)
```

This is the kind of query that is structurally awkward in parallel-provenance designs and trivial with triple terms.

### 3.6 Retraction Semantics — The Killer Feature

The most consequential property of triple terms for code knowledge graphs is **first-class retraction**. In an incremental pipeline (see §9.2), a derived triple becomes invalid when its source changes. With triple terms, the dependency between a derivation and its source is itself a triple in the graph:

```turtle
<#claim_b> rdf:reifies << <#functionA> lsp-x:calls <#functionB> >> ;
    repolex:resolvedBy "scip-java" ;
    repolex:derivedFromBlob "blob:sha256:...abc123" .
```

When the blob changes, every claim referencing it can be retracted with a single SPARQL UPDATE. Without triple terms, you cannot refer to the derived triple as a first-class subject, so you cannot annotate it with its retraction key, so you have to maintain a parallel index of "what triples depend on what blobs." With triple terms, the dependency is in the graph and can be queried, validated, and traversed like anything else.

This is the property that makes incremental aggregation over RDF practical, and it is unique to RDF 1.2.

### 3.7 Implementation Notes

- Oxigraph 0.5.x supports triple terms by default in Python, JS, and CLI bindings; it is opt-in via feature flag in the Rust crate. repolex/rlex use the default-on bindings.
- Apache Jena 6.0.0 supports triple terms in all parsers (Turtle, TriG, N-Triples, N-Quads). Reading and writing both work.
- RDF4J 5.3.0-M2 has a `Triple` class predating RDF 1.2; alignment is conservative. Watch for the next major release.
- QLever does *not* yet support RDF 1.2. This is the gating concern for any future scale-out path that requires triple-term provenance.

The repolex code ontology uses triple terms for: call edges (this section), security findings (§5), extraction provenance (§9.1), and inferred edges (§9.2).

## §4 Alignments

*[TR1P.L3X — to draft]*

- PROV-O for provenance (Activity, Agent, Entity, wasGeneratedBy, wasAssociatedWith)
- SKOS for tag systems and concept hierarchies
- FaBiO for citations (when code references papers)
- schema.org SoftwareSourceCode for repo-level metadata
- DCTERMS for publication / licensing
- The repolex code ontology as a first-class member of this stack

## §5 SHACL Shapes as the Validation Contract

### 5.1 The Two-Layer Discipline

This specification uses a two-layer discipline for constraints:

1. **OWL** is the source of truth for the *meaning* of types and properties — domains, ranges, subclass relationships, restrictions on cardinality and value sets.
2. **SHACL** is the source of truth for *validation* — generated from OWL via a deterministic translator. Shape files are derived artifacts, not hand-edited.

This is the inverse of the more common pattern, in which SHACL shapes are written by hand and OWL classes are inferred or omitted entirely. The reasons we prefer OWL-as-source:

- **One specification, two outputs.** OWL gives you reasoning (subsumption, classification, restriction-based inference). SHACL gives you validation. Generating SHACL from OWL means you get both for free, with one source.
- **Round-tripping.** A constraint expressed as `owl:Restriction` or `owl:oneOf` can be machine-translated into a `sh:NodeShape` with the appropriate property constraints. The reverse translation is harder because SHACL is more expressive than OWL in some directions.
- **Documentation parity.** OWL classes are the natural unit of ontology documentation. SHACL shapes generated from OWL inherit the labels and comments, so the validation messages are meaningful by default.

The git-lex implementation generates `*-shapes.ttl` from `*.ttl` ontology files via the `git-lex sync` pipeline. Hand-editing the shapes file is an anti-pattern; if a shape needs to change, the change goes in the OWL.

### 5.2 What a Validation-as-Contract Looks Like

A SHACL shape generated from this ontology has three jobs:

1. **Reject malformed data.** A `Function` without a `name` is malformed; the shape rejects it.
2. **Document the schema.** The shape file is the most precise description of what a valid instance looks like, suitable for tooling and humans alike.
3. **Compose with other shapes.** Multiple shape files can be loaded into a validator and applied together. A repolex graph that also imports PROV-O shapes can be validated against both.

A minimal shape for a `FunctionDefinition`:

```turtle
@prefix sh:    <http://www.w3.org/ns/shacl#> .
@prefix ast-x: <https://repolex.ai/ontology/repolex/ast-extension/> .
@prefix xsd:   <http://www.w3.org/2001/XMLSchema#> .
@prefix rdfs:  <http://www.w3.org/2000/01/rdf-schema#> .

ast-x:FunctionDefinitionShape a sh:NodeShape ;
    sh:targetClass ast-x:FunctionDefinition ;
    sh:property [
        sh:path rdfs:label ;
        sh:datatype xsd:string ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
        sh:message "FunctionDefinition must have exactly one rdfs:label."
    ] ;
    sh:property [
        sh:path ast-x:cyclomaticComplexity ;
        sh:datatype xsd:integer ;
        sh:maxCount 1 ;
        sh:minInclusive 0 ;
        sh:message "cyclomaticComplexity must be a non-negative integer."
    ] ;
    sh:property [
        sh:path ast-x:lineCount ;
        sh:datatype xsd:integer ;
        sh:maxCount 1 ;
        sh:minInclusive 0
    ] .
```

Three properties, one with cardinality `[1,1]` (the label), two optional. A validator running this shape against a graph will produce one violation per malformed function with a precise message and a pointer to the offending node.

### 5.3 Validation as Round-Trip Insurance

Generated SHACL shapes serve a particular function in the repolex pipeline: they are the integrity check on the round-trip between source code and RDF. After every parse:

```
source code → tree-sitter AST → repolex extraction → RDF triples → SHACL validation → store
```

Validation runs on the *output* of extraction. A failing validation means either:

- The extractor produced a malformed node (bug in the parser)
- The ontology constrained the wrong thing (bug in the OWL)

Both are fixable. Both are detectable. Both are caught at the boundary, not at query time. Without SHACL, errors propagate into the store and surface as silent query results — a function with no name, an edge with no target, a class with a string where it should be a URI. With SHACL, the error is a validation failure with a row, a column, and a message.

The TRIPLEFORCE squad's git-lex pipeline runs SHACL on every commit via a pre-commit hook. The discipline: if the SHACL fails, the commit fails. The graph never enters an invalid state.

### 5.4 Roadmap to SHACL 1.2 Rules and Node Expressions

SHACL 1.2 (currently FPWD, projected REC late 2026 or early 2027) introduces two capabilities that change what shapes can do:

**Rules** add forward-chaining inference. A rule fires when its condition matches and asserts new triples. This makes SHACL a *production system*, not just a validator. Repolex has many "rules" today that live in imperative extraction code: "if a function calls more than 10 other functions, mark it as a hub"; "if a class name ends in `Test` and inherits from a test framework class, mark it as a test"; "if an import resolves to an external package, attach the package coordinates." Each of these is currently a SPARQL UPDATE in `enrich.py`. With SHACL Rules, each becomes a declarative shape:

```turtle
ast-x:HubFunctionRule a sh:NodeShape ;
    sh:targetClass ast-x:FunctionDefinition ;
    sh:rule [
        a sh:TripleRule ;
        sh:condition [
            sh:property [
                sh:path lsp-x:callsCount ;
                sh:minInclusive 10
            ]
        ] ;
        sh:subject sh:this ;
        sh:predicate ast-x:isHubFunction ;
        sh:object true
    ] .
```

**Node Expressions** (FPWD 2026-01-08) introduce compositional value computation. A property's value can be computed from the data, not just constrained. This collapses the distinction between "schema" and "logic" — the same shape file can both validate and derive. The canonical example (cited in §9) is date intelligence in `fm:` namespace properties: detect that a property name matches `.*[Dd]ate.*` and apply a typed-literal constraint or coercion. Today this lives in extraction heuristics; with Node Expressions it becomes a shape declaration.

The repolex roadmap is to migrate enrichment rules and extraction heuristics into SHACL 1.2 shapes once oxigraph (or another RDF 1.2 store) ships SHACL 1.2 support. The contract is:

- If it currently looks like a special case in extraction, it should be a shape in the spec.
- Shapes are inspectable, modifiable, and queryable. Imperative code is none of those things.
- The migration is gradual: each shape that moves from code to spec is a small pull request, not a big rewrite.

This is the structural reason the spec layers OWL → generated SHACL today: when the SHACL can also carry rules and computed properties, the same generation pipeline upgrades for free.

## §6 Worked Examples

Each example follows the same pattern: source code → key triples → a SPARQL query that demonstrates a real use case. We use abbreviated triples for readability; the full RDF is generated by the repolex parser and lives in the test fixtures.

### 6.1 Python — Class with Methods

**Source** (`example.py`):

```python
class Greeter:
    """Greets people by name."""

    def __init__(self, name: str):
        self.name = name

    def greet(self) -> str:
        return f"Hello, {self.name}!"
```

**Triples (abbreviated):**

```turtle
<file:example.py#Greeter_0_0_8_42>
    a ast-x:ClassDefinition ;
    rdfs:label "Greeter" ;
    ast-x:lineCount 9 ;
    ast-x:filePath "example.py" .

<file:example.py#Greeter.__init___3_4_4_25>
    a ast-x:FunctionDefinition ;
    rdfs:label "__init__" ;
    ast-x:cyclomaticComplexity 1 ;
    ast-x:lineCount 2 ;
    ast-x:enclosingClass <file:example.py#Greeter_0_0_8_42> .

<file:example.py#Greeter.greet_6_4_7_42>
    a ast-x:FunctionDefinition ;
    rdfs:label "greet" ;
    ast-x:cyclomaticComplexity 1 ;
    ast-x:lineCount 2 ;
    ast-x:enclosingClass <file:example.py#Greeter_0_0_8_42> .
```

**Query — list all methods of a given class:**

```sparql
SELECT ?methodName ?complexity WHERE {
    ?class a ast-x:ClassDefinition ;
           rdfs:label "Greeter" .
    ?method a ast-x:FunctionDefinition ;
            ast-x:enclosingClass ?class ;
            rdfs:label ?methodName ;
            ast-x:cyclomaticComplexity ?complexity .
}
```

### 6.2 Rust — Impl Block with Trait Implementation

**Source** (`example.rs`):

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Greeter {
    name: String,
}

impl Greet for Greeter {
    fn greet(&self) -> String {
        format!("Hello, {}!", self.name)
    }
}
```

**Triples (abbreviated):**

```turtle
<file:example.rs#Greet_0_0_2_1>
    a ast-x:Interface ;
    rdfs:label "Greet" .

<file:example.rs#Greeter_4_0_6_1>
    a ast-x:ClassDefinition ;
    rdfs:label "Greeter" .

<file:example.rs#impl_8_0_12_1>
    a ast-x:ImplBlock ;
    lsp-x:implementsInterface <file:example.rs#Greet_0_0_2_1> ;
    ast-x:enclosingClass <file:example.rs#Greeter_4_0_6_1> .

<file:example.rs#Greeter.greet_9_4_11_5>
    a ast-x:FunctionDefinition ;
    rdfs:label "greet" ;
    ast-x:enclosingClass <file:example.rs#Greeter_4_0_6_1> ;
    lsp-x:implementsMethod <file:example.rs#Greet.greet_1_4_1_29> .
```

**Query — find all classes that implement a given trait:**

```sparql
SELECT ?className WHERE {
    ?impl a ast-x:ImplBlock ;
          lsp-x:implementsInterface ?trait ;
          ast-x:enclosingClass ?class .
    ?trait rdfs:label "Greet" .
    ?class rdfs:label ?className .
}
```

### 6.3 TypeScript — Class with Type Parameters

**Source** (`example.ts`):

```typescript
interface Greeter<T> {
    greet(target: T): string;
}

class StringGreeter implements Greeter<string> {
    greet(target: string): string {
        return `Hello, ${target}!`;
    }
}
```

**Triples (abbreviated):**

```turtle
<file:example.ts#Greeter_0_0_2_1>
    a ast-x:Interface ;
    rdfs:label "Greeter" ;
    ast-x:typeParameter <file:example.ts#Greeter.T> .

<file:example.ts#Greeter.T>
    a ast-x:TypeParameter ;
    rdfs:label "T" .

<file:example.ts#StringGreeter_4_0_8_1>
    a ast-x:ClassDefinition ;
    rdfs:label "StringGreeter" ;
    lsp-x:implementsInterface <file:example.ts#Greeter_0_0_2_1> ;
    lsp-x:typeArgument [
        ast-x:typeParameter <file:example.ts#Greeter.T> ;
        ast-x:resolvedType "string"
    ] .
```

**Query — find all concrete instantiations of a generic interface:**

```sparql
SELECT ?className ?typeArg WHERE {
    ?class a ast-x:ClassDefinition ;
           rdfs:label ?className ;
           lsp-x:implementsInterface ?iface ;
           lsp-x:typeArgument ?ta .
    ?iface rdfs:label "Greeter" .
    ?ta ast-x:resolvedType ?typeArg .
}
```

### 6.4 Security — Unicode Steganography in a String Literal

**Source** (`config.py`):

```python
API_KEY = "sk-prod-12345"  # contains a zero-width joiner: U+200D between "sk" and "-prod"
```

**Triples (abbreviated):**

```turtle
<file:config.py#API_KEY_0_0_0_30>
    a ast-x:VariableDeclaration ;
    rdfs:label "API_KEY" ;
    ast-x:initializer <file:config.py#API_KEY_value> .

<file:config.py#API_KEY_value>
    a ast-x:StringLiteral ;
    ast-x:textValue "sk\u200D-prod-12345" .

<file:config.py#API_KEY_value> security:hasUnicodeAnomaly <#anomaly_001> .

<#anomaly_001>
    a security:UnicodeAnomaly ;
    security:anomalyType "ZeroWidthJoiner" ;
    security:codePoint "U+200D" ;
    security:offsetInString 2 ;
    security:detectedBy "homoglyph_scanner_v2" ;
    security:detectedAt "2026-04-07T12:00:00Z"^^xsd:dateTime ;
    security:confidence "0.95"^^xsd:decimal .

# Provenance via triple term
<#claim_001>
    rdf:reifies << <file:config.py#API_KEY_value> security:hasUnicodeAnomaly <#anomaly_001> >> ;
    prov:wasGeneratedBy <#scan_run_42> ;
    prov:wasAssociatedWith <#agent_steg_scanner> .
```

**Query — find all string literals with steganographic anomalies, with their detection method:**

```sparql
SELECT ?file ?codePoint ?detector ?confidence WHERE {
    ?literal a ast-x:StringLiteral ;
             security:hasUnicodeAnomaly ?anomaly .
    ?anomaly security:codePoint ?codePoint ;
             security:detectedBy ?detector ;
             security:confidence ?confidence .
    ?literal ast-x:filePath ?file .
    FILTER (?confidence > 0.8)
}
ORDER BY DESC(?confidence)
```

### 6.5 Cross-Cutting — Find All Functions That Call X Across All Languages

This is the query that justifies the substrate. The same call edge predicate (`lsp-x:calls`) is used across all language extractors, so a single SPARQL query traverses Python, Rust, TypeScript, Java, and any other language with a repolex extractor.

**Query:**

```sparql
PREFIX ast-x: <https://repolex.ai/ontology/repolex/ast-extension/>
PREFIX lsp-x: <https://repolex.ai/ontology/repolex/lsp-extension/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?caller ?callerLanguage ?callerFile WHERE {
    ?target a ast-x:FunctionDefinition ;
            rdfs:label "executeSelect" .
    ?caller lsp-x:calls ?target .
    ?caller a ast-x:FunctionDefinition ;
            ast-x:filePath ?callerFile .
    ?callerFile ast-x:language ?callerLanguage .
}
ORDER BY ?callerLanguage ?callerFile
```

The query does not need to know which extractor produced any given edge. The vocabulary is shared across languages, the federation is built into RDF, and the answer composes across the entire loaded store. A property graph cannot do this without a custom join layer per language.

### 6.6 What These Examples Demonstrate

- **Same vocabulary, different languages.** Python classes, Rust impl blocks, and TypeScript generics are all expressed via the same `ast-x:` and `lsp-x:` predicates. Cross-language queries are trivial.
- **Provenance is queryable.** §6.4 shows a security finding annotated with detection method, confidence, and detection run. The provenance is a triple term, queryable in the same graph as the finding.
- **The query language is standard.** Every example above is plain SPARQL 1.1, runnable on any conforming engine. There is no DSL, no proprietary query language, and no per-tool dialect.
- **Federation is free.** Loading multiple repos into the same store enables cross-repo queries by default. There is no ETL pipeline to write.

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

### 7.5.1 Why SCIP Specifically

SCIP (Source Code Index Protocol, Sourcegraph) is the de facto interchange format for code intelligence as of 2026. It is:

- **Protobuf-based**, so compact and language-neutral
- **Indexer-agnostic**: indexers exist for Java, JavaScript/TypeScript, Rust (built into rust-analyzer), Go, Python, Kotlin, Scala, C++, C#, Ruby, Dart — twelve of the languages we currently care about
- **Symbol-stable**: the SCIP symbol grammar produces deterministic, indexer-agnostic IDs across runs and across machines
- **Cross-repo by design**: external symbols carry their package coordinates in the symbol string itself

For repolex, SCIP solves the "fast batch enrichment" problem better than live LSP: an indexer runs once per project, produces a Protobuf file, and downstream queries are O(1) string lookups instead of O(N) LSP round-trips. The SCIP-as-LSP-alternative research and the SCIP→RDF mapping draft (both in the TRIPLEFORCE squad repo) explore this in depth.

The relevant question for *this* spec: how does the repolex code ontology absorb SCIP without forking?

### 7.5.2 The Composition Strategy

The spec treats SCIP as one input source among several. The pipeline is:

```
SCIP Protobuf  ──┐
tree-sitter AST ─┼─→ repolex extraction ─→ RDF triples ─→ store
LSP responses  ──┘
```

All three sources produce triples in the *same* vocabulary. SCIP-derived edges and tree-sitter-derived edges are indistinguishable to downstream queries unless they ask about provenance — in which case the triple-term annotations from §3 carry the source.

The key design decision is that SCIP symbols become **a property of repolex nodes**, not a parallel URI scheme. A function defined at `path/to/file.java:42` has a canonical repolex URI based on its position; if SCIP also identifies that function via a SCIP symbol string, the SCIP symbol is attached as `repolex:scipSymbol`:

```turtle
<https://repolex.ai/r/apache/jena/commit/abc123/jena-arq/src/main/java/org/apache/jena/sparql/engine/QueryExecutionBase.java#executeSelect_42_4_67_5>
    a ast-x:FunctionDefinition ;
    rdfs:label "executeSelect" ;
    repolex:scipSymbol "scip-java maven org.apache.jena jena-arq 4.10.0 org/apache/jena/sparql/engine/QueryExecutionBase#executeSelect()." ;
    ast-x:filePath "jena-arq/src/main/java/org/apache/jena/sparql/engine/QueryExecutionBase.java" .
```

Two URIs, one node. The position-keyed URI matches what tree-sitter and LSP-based extractors produce for the same function, so SCIP-derived data merges naturally with existing data. The SCIP symbol becomes a queryable property.

### 7.5.3 Why Not a Parallel `scip:` IRI Scheme?

Two URI schemes for the same thing is a footgun. If SCIP-derived nodes use `<scip:java/maven/org.apache.jena/...>` and tree-sitter-derived nodes use `<file://path/to/file#position>`, every cross-source query needs an `OWL:sameAs` traversal to bridge them. The URIs split the graph.

Using one canonical URI (position-keyed, matching the existing repolex scheme) and exposing the SCIP symbol as a property keeps the graph unified. A query that wants to find a function by SCIP symbol does so by property:

```sparql
SELECT ?func WHERE {
    ?func repolex:scipSymbol "scip-java maven org.apache.jena jena-arq 4.10.0 org/apache/jena/sparql/engine/QueryExecutionBase#executeSelect()." .
}
```

A query that wants to find the same function by file path does so by URI:

```sparql
SELECT ?func WHERE {
    ?func ast-x:filePath "jena-arq/src/main/java/org/apache/jena/sparql/engine/QueryExecutionBase.java" ;
          rdfs:label "executeSelect" .
}
```

Both queries return the same node. There is no fork.

### 7.5.4 Worked Example — Cross-Repo Symbol Resolution

The killer query that SCIP enables: find all callers of a specific upstream function across every loaded repo.

Suppose you have loaded `apache/jena`, `apache/jena-fuseki2`, and `eclipse/rdf4j`. All three depend on or interact with `org.apache.jena.sparql.engine.QueryExecutionBase`. A traditional file-path query cannot find them because the function is defined in one repo and called in many, and each call site has a different file path. A SCIP-symbol query handles it in one BGP:

```sparql
PREFIX repolex: <https://repolex.ai/ontology/repolex/>
PREFIX lsp-x:   <https://repolex.ai/ontology/repolex/lsp-extension/>
PREFIX ast-x:   <https://repolex.ai/ontology/repolex/ast-extension/>

SELECT ?caller ?callerRepo ?callerFile WHERE {
    ?target repolex:scipSymbol "scip-java maven org.apache.jena jena-arq 4.10.0 org/apache/jena/sparql/engine/QueryExecutionBase#executeSelect()." .
    ?caller lsp-x:calls ?target ;
            ast-x:filePath ?callerFile .
    ?callerFile repolex:repository ?callerRepo .
}
ORDER BY ?callerRepo ?callerFile
```

The deterministic SCIP symbol is the join key. No federation layer, no ETL — the graph composes natively because all three repos use the same vocabulary and the same symbol-as-property pattern.

### 7.5.5 Composing With LSIF, Stack Graphs, and Future Formats

The same strategy applies to other code-intelligence formats:

- **LSIF** (LSP Indexed Format): older Microsoft format, similar role to SCIP. Add `repolex:lsifId` as a property.
- **Stack Graphs**: tree-sitter-based incremental code intelligence from GitHub. Add `repolex:stackGraphSymbol`.
- **Future formats**: any new identifier scheme can be added as a property without changing the URI scheme or the existing queries.

The repolex code ontology stays canonical; external identifiers are queryable annotations on canonical nodes. This is the inverse of the property-graph world, where every tool ships its own node IDs and federation requires custom translators.

### 7.5.6 Summary

SCIP is absorbed, not adopted wholesale. The mapping is:

| SCIP construct | repolex representation |
|---|---|
| SCIP `Document` | `repolex:File` (existing, augmented with SCIP language metadata) |
| SCIP `SymbolInformation` | `ast-x:FunctionDefinition` / `ClassDefinition` / etc. (typed via kind mapping), with `repolex:scipSymbol` property |
| SCIP `Occurrence` | `lsp-x:Reference` with position, role bitset, and `lsp-x:enclosingFunction` |
| SCIP `relationships[]` | `lsp-x:implementsInterface`, `lsp-x:typeOf`, etc. — existing predicates |
| SCIP `external_symbols` | cross-repo refs with `repolex:scipSymbol` and `repolex:packageName/Version` |
| SCIP `enclosing_range` | `lsp-x:enclosingFunction` (the dataflow analysis win — eliminates expensive parent-traversal queries) |

The full mapping is documented in the [SCIP→RDF mapping spec](experiment/scip-to-repolex.md), which is itself a `lab:Experiment` artifact in this repository. That document is the worked validation that the absorption is mechanical and lossless for the data we care about.

The point of §7.5 is the principle: **the repolex code ontology is the canonical layer; SCIP and other code-intelligence formats are absorbed via property annotations, not URI forks.** This is what "RDF substrate, multiple input formats" means in practice.

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
