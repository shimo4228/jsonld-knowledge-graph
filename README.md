# jsonld-knowledge-graph

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that designs and ships a companion **JSON-LD knowledge graph** (`graph.jsonld`) next to `llms.txt` for projects whose concept-level structure is stable across releases.

Encodes domain entities and relationships as [schema.org](https://schema.org/)-compatible triples so LLMs (ChatGPT / Perplexity / Gemini / Claude) can cite the structure machine-readably — beyond what prose alone can convey.

## When to use

Apply when **all** of the following hold:

- The project has stable concept-level structure (a matrix, ordered hierarchy, phase-to-skill binding, layered architecture, etc.)
- The structure does **not** change across `vX.Y.Z` releases
- You have empirical evidence that LLMs answer relationship questions poorly from prose alone
- The project already has `llms.txt` + `llms-full.txt` (Answer.AI standard)

If your project is a single-purpose linear codebase or has internally churning structure, **don't** use this — `graph.jsonld` would become an editing burden.

## Install

### Claude Code

```bash
# Copy skill into your global skills directory
cp -r skills/jsonld-knowledge-graph ~/.claude/skills/jsonld-knowledge-graph
```

No Python dependencies. The skill is documentation-only; verification commands inside use `python3 -m json.tool` and optional `uvx --from pyld`.

### SkillsMP

```bash
/skills add shimo4228/jsonld-knowledge-graph
```

## How it works

1. **When-to-use gate** — the skill checks four conditions (stable structure + release-stability + LLM-citation evidence + existing llms.txt) before any design work.
2. **9 reusable design moves** — dual `@type`, language-tagged literals, schema-absence enforcement, cross-graph `@id` reuse, volatile-state exclusion, matrix-as-paired-edges, root `Dataset` node, reading-order block, hub-and-spoke reverse-link.
3. **Companion file wiring** — surfaces `graph.jsonld` to crawlers via `llms.txt` reading-order block, `llms-full.txt` question-form H2, and a README AI-facing reading order `<details>` block.
4. **Maintenance contract** — explicit triggers for *when to edit* (new EcosystemRepo / Concept / ResearchLine) and *when NOT to edit* (routine releases, version bumps, ADR count changes).

## Key concept: schema absence enforces invariants

The skill emphasizes that the strongest way to prevent a wrong relationship from being encoded is to **not define an edge type for it**. For example, if three research lines must remain siblings (never dependencies), the shared vocabulary defines `siblingOf` but deliberately does **not** define `dependsOn`. The vocabulary itself becomes a structural commitment.

Similarly, by not putting `version` / `count` / `vX.Y.Z` field names in the schema, routine releases cannot leak volatile state into the graph even by accident.

## What this skill does NOT do

| Concern | Use this instead |
|---|---|
| llms.txt / llms-full.txt prose design, navigator wording, GEO optimization | [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer) |
| Project doc role overlap / freshness audit | [context-sync](https://github.com/shimo4228/context-sync) |
| File-level architecture maps (CODEMAPS) | [claude-skill-update-codemaps](https://github.com/shimo4228/claude-skill-update-codemaps) (if available) |
| Article / blog post writing | `article-writing` / [claude-skill-writing-ecosystem](https://github.com/shimo4228/claude-skill-writing-ecosystem) |

## Related skills

- [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer) — writes `llms.txt` / `llms-full.txt` / FAQ / glossary; the navigator wording for `graph.jsonld` lives there
- [context-sync](https://github.com/shimo4228/context-sync) — audits drift between `graph.jsonld` and CODEMAPS during the Maintain phase
- [search-first](https://github.com/shimo4228/search-first) — research-before-building workflow

## Verification

After editing `graph.jsonld`:

```bash
# JSON syntax
python3 -m json.tool < graph.jsonld > /dev/null

# JSON-LD expansion + N-Quads triple count
uvx --quiet --from pyld python3 -c "
from pyld import jsonld
import json
doc = json.load(open('graph.jsonld'))
expanded = jsonld.expand(doc)
nquads = jsonld.to_rdf(doc, {'format': 'application/n-quads'})
print(f'{len(expanded)} nodes / {len([l for l in nquads.strip().split(chr(10)) if l])} triples')
"

# Volatile state (should be empty)
grep -E '"version"|"versionNumber"|"adrCount"|v[0-9]+\.[0-9]+' graph.jsonld
```

Manual checks: [JSON-LD playground](https://json-ld.org/playground/), [schema.org validator](https://validator.schema.org/), and LLM-citation probing after crawler refresh (1–2 weeks post-push).

## About this skill

This skill is a **component skill of the [Authorship Strategy](https://github.com/shimo4228/authorship-strategy) research line** ([DOI 10.5281/zenodo.20263316](https://doi.org/10.5281/zenodo.20263316)) maintained by [@shimo4228](https://github.com/shimo4228). It is the operational form of the *concept-form graph* half of the **dual entry point** that [ADR-0006](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0006-llm-first-ingest-dual-entry-points.md) normatively requires for any framework-governed artifact. Its companion is [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer), which operationalizes the *prose-form navigator* half; per ADR-0006, deploying only one half leaves the strategy one-lunged — each entry point addresses a distinct LLM-mediated reader sub-population the other cannot reach.

The skill is published alongside the broader research program: three agent-design lines ([Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle) — mechanism, [DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) — disposition, [DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118); [Agent Attribution Practice](https://github.com/shimo4228/agent-attribution-practice) — accountability practice, [DOI 10.5281/zenodo.19652013](https://doi.org/10.5281/zenodo.19652013)) and two cross-cutting lines (Authorship Strategy itself; [Attention, Not Self](https://github.com/shimo4228/attention-not-self) — Buddhist Abhidharma meets computational phenomenology, [DOI 10.5281/zenodo.20262112](https://doi.org/10.5281/zenodo.20262112)).

## License

MIT. See [LICENSE](LICENSE).
