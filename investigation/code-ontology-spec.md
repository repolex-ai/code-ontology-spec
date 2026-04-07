---
title: "Defining an RDF code ontology for the AI era"
tags: [code-ontology, spec, rdf, repolex]
lab.investigation.researchQuestion: "What does a complete RDF vocabulary for code knowledge graphs look like, and how does it compose with existing formats (SCIP, LSIF, CPG, CodeQL)?"
lab.investigation.investigationStatus: "active"
lab.investigation.lead: "[[tr1p-l3x]]"
lab.investigation.contributors: ["[[m4rq]]", "[[spacegoat]]"]
---

# Investigation: Defining an RDF Code Ontology for the AI Era

## The Question

What does a complete RDF vocabulary for code knowledge graphs look like, and how does it compose with existing formats (SCIP, LSIF, CPG, CodeQL)?

## Why Now

Three things converged in early 2026 that make this the right moment:

1. **No dominant RDF code ontology exists.** Every major code-analysis tool uses a different property-graph or custom format. The semantic web's strongest claim — federation, composition, shared vocabularies — has been ignored by the code-tools world for two decades.

2. **AI agents need structured code memory.** LLMs are eating the developer workflow, and "give the agent persistent code memory" is a problem everyone is solving with vector embeddings or ad-hoc property graphs. RDF is the right substrate for this and nobody is shipping it.

3. **RDF 1.2 is here.** Triple terms + `rdf:reifies` give us a way to attach provenance, confidence, and inference traces to code edges directly — no clunky reification, no parallel metadata graphs. This is exactly what we need for "this call edge was extracted by tree-sitter v0.25 with confidence 1.0" semantics.

The TRIPLEFORCE squad has been building this vocabulary for repolex for the last six months. It works. It validates. It scales (currently at 1.2M+ triples on real codebases). Now it's time to publish the spec so others can adopt and federate.

## Hypothesis

A small layered RDF vocabulary (ast-base, ast-x, lsp-x, security, lex-upper) plus alignments to existing standards (PROV-O for provenance, SKOS for tag systems, FaBiO for citations, schema.org for projects) is sufficient to capture the structural, semantic, and metadata layers of code knowledge graphs. The vocabulary should compose with SCIP (the dominant code interchange format) so existing parsers can populate the graph without rewriting their pipelines.

## Approach

Build the spec through worked examples first, then generalize. For each language (Python, Rust, TypeScript) we already have working parsers in repolex. The spec should reverse-engineer the vocabulary we've actually shipped and prove it generalizes.

For SCIP composability, write a mapping from SCIP symbol grammar to repolex IRI scheme. SpaceG.O.A.T. is drafting this in parallel.

## Status

Draft in progress. See `synthesis/code-ontology-spec.md` for the current state of the spec itself.
