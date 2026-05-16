# Changelog

All notable changes to this project will be documented in this file.

## [0.1.0] — 2026-05-16

Initial public release.

### What it does

A Claude Code skill that designs and ships a companion JSON-LD knowledge graph
(`graph.jsonld`) next to `llms.txt` for projects with stable concept-level
structure. Encodes domain entities and relationships as schema.org-compatible
triples so LLM-based search engines (ChatGPT / Perplexity / Gemini / Claude)
can cite the structure machine-readably, beyond what prose alone conveys.

### Components

- `SKILL.md` — when-to-use gate, 9 reusable design moves, schema vocabulary
  design criteria, cross-graph `@id` discipline, bilingual strategy, companion
  file wiring, verification workflow, maintenance contract.
- `inspiration.md` — origin story and canonical implementation pointers (kept
  separate from `SKILL.md` so the skill remains portable to other projects).

### 9 Design Moves

1. Dual `@type` on every node (custom + schema.org base)
2. Language-tagged literals for bilingual (`@language` per `@value`)
3. Schema absence enforces forbidden relationships
4. Cross-graph `@id` reuse for triple merge
5. Volatile state excluded at schema level
6. Matrix encoding via paired complementary edges
7. Root `Dataset` node for graph self-description
8. Reading-order block in `llms.txt` for AI entry point
9. Reverse-link from line repos to hub graph (hub-and-spoke topology)

### Requirements

- No runtime dependencies. Verification commands use `python3` (stdlib) and
  optional `uvx --from pyld` for JSON-LD expansion.

### Relationship to companion skills

| Skill | Role | When |
|---|---|---|
| `llms-txt-writer` | prose / navigator | After deciding to add `graph.jsonld`, wire it via `llms.txt` |
| `context-sync` | drift audit | Maintain phase — checks `graph.jsonld` vs CODEMAPS |
| `update-codemaps` | file-level docs | Complementary, never overlapping concerns |
