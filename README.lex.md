# git-lex — lab kit

This repo is managed by [git-lex](https://github.com/repolex-ai/git-lex) — git extensions for knowledge graphs.

## Quick Start

```bash
git lex create <type>    # Create a new document (types: experiment, survey, hypothesis, investigation, deliberation, synthesis, researchbranch)
git lex save "message"   # Add + commit (extracts automatically)
git lex sync              # Build/update the knowledge graph
git lex query "SPARQL..." # Query the knowledge graph
git lex status            # Show extraction status
```

## Commands

| Command | What it does |
|---|---|
| `git lex create <type>` | Scaffold a new document. Valid types: experiment, survey, hypothesis, investigation, deliberation, synthesis, researchbranch |
| `git lex save "msg"` | Stage all changes, commit, extract frontmatter |
| `git lex sync` | Build the SPARQL knowledge graph from git + extractions |
| `git lex query "..."` | Run a SPARQL query against the knowledge graph |
| `git lex status` | Show which files have been extracted |
| `git lex log` | Show commit history |
| `git lex llm list` | Show files needing LLM extraction |
| `git lex llm extract <file> --model <id>` | Extract entities via LLM |

## Writing Documents

Documents use YAML frontmatter with flat dot notation: `kit.class.property`

```yaml
---
lab.memory.confidence: "certain"
lab.memory.source: "observation"
lab.memory.category: "fact"
---

Your content here. Use @mentions and [[wikilinks]] for relationships.
```

See `__ClassName.md` files in each folder for available properties and SHACL-derived constraints.

## @mentions and [[wikilinks]]

Reference other agents and documents naturally in your text:

- `@agentname` — creates a `lex:mentions` relationship
- `[[document-title]]` — creates a `lex:linksTo` relationship

These are extracted automatically from document bodies and commit messages.

## lab Kit — Document Types

### Experiment

Create: `git lex create experiment`

| Property | Type | Description |
|---|---|---|
| protocol | string | What was done — method, steps, configuration. |
| result | string | What was observed — raw output, measurements, data. |
| conclusion | string | Interpretation of the result — what it means for the hypothesis. |
| experimentStatus | string | Status: 'planned', 'running', 'complete', 'failed', 'abandoned'. |
| tests | reference | The hypothesis this experiment is designed to test. |
| partOf | reference | The investigation this experiment contributes to. |
| replicates | reference | An earlier experiment this one attempts to reproduce. |
| runBy | reference | The agent who ran this experiment. |
| commitRef | string | Git commit SHA where this experiment was run. The commit IS the reproducible artifact — checkout this SHA to reproduce. |
| dataset | reference | Link to dataset(s) used or produced by this experiment. |

### Survey

Create: `git lex create survey`

| Property | Type | Description |
|---|---|---|
| scope | string | What was surveyed — which sources, time range, search terms. |
| findings | string | What the survey found — key takeaways, gaps, patterns. |
| sources-list | string | List of sources surveyed: URLs, papers, ontologies, repos. |
| surveyStatus | string | Status: 'planned', 'in-progress', 'complete'. |
| informs | reference | The investigation this survey informs. |

### Hypothesis

Create: `git lex create hypothesis`

| Property | Type | Description |
|---|---|---|
| claim | string | The falsifiable statement being evaluated. |
| hypothesisStatus | string | Lifecycle: 'proposed', 'active', 'supported', 'refuted', 'revised', 'abandoned'. |
| revisedBy | reference | A later hypothesis that refines or replaces this one. |
| addresses | reference | The investigation this hypothesis is formulated to answer. |
| confidence | string | Current confidence level: 'speculative', 'plausible', 'likely', 'supported', 'refuted'. |

### Investigation

Create: `git lex create investigation`

| Property | Type | Description |
|---|---|---|
| researchQuestion | string | The question this investigation is trying to answer. |
| investigationStatus | string | Status: 'open', 'active', 'paused', 'concluded', 'abandoned'. |
| lead | reference | The agent leading this investigation. |
| contributors | reference | Other agents contributing to this investigation. |
| parentInvestigation | reference | The broader investigation this one is a sub-thread of. |
| concludedBy | reference | The synthesis document that concludes this investigation. |

### Deliberation

Create: `git lex create deliberation`

| Property | Type | Description |
|---|---|---|
| deliberationStatus | string | Status: 'open', 'converging', 'concluded', 'deadlocked'. |
| outcome | string | How the deliberation ended: 'consensus', 'dissensus', 'deferred', 'merged'. |
| regarding | reference | The hypothesis or investigation this deliberation is about. |
| producedSynthesis | reference | The synthesis this deliberation concluded with, if any. |

### Synthesis

Create: `git lex create synthesis`

| Property | Type | Description |
|---|---|---|
| synthesizedClaim | string | The central claim this synthesis makes. Should be a precise, citable statement. |
| synthesisStatus | string | Status: 'draft', 'review', 'published', 'superseded'. |
| evidenceStrength | string | How strong is the evidence: 'weak', 'moderate', 'strong', 'conclusive'. |
| concludes | reference | The investigation this synthesis concludes. |
| basedOn | reference | Experiments and surveys this synthesis draws from. |
| supports | reference | A hypothesis this synthesis supports. |
| refutes | reference | A hypothesis this synthesis refutes. |
| supersedes | reference | An earlier synthesis this one replaces. |
| authors | reference | The agents who wrote this synthesis. |

### ResearchBranch

Create: `git lex create researchbranch`

| Property | Type | Description |
|---|---|---|
| branchName | string | The git branch name — also the hypothesis identifier. |
| embodiesHypothesis | reference | The hypothesis being developed on this branch. |

## Querying

Auto-injected prefixes: `git:`, `fm:`, `lex:`, `lex-o:`, `lab:`

```sparql
# List all documents by type
SELECT ?name ?type WHERE {
  GRAPH ?g { ?doc fm:lab.type ?type ; fm:title ?name }
}
```
