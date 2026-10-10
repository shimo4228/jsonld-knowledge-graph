# jsonld-knowledge-graph

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/jsonld-knowledge-graph)

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that designs and ships a companion **JSON-LD knowledge graph** (`graph.jsonld`) next to `llms.txt` for projects whose concept-level structure is stable across releases.

Encodes domain entities and relationships as [schema.org](https://schema.org/)-compatible triples so LLMs (ChatGPT / Perplexity / Gemini / Claude) get the relationships as explicit statements and can resolve the project's entities (tell which project, paper or concept a name refers to), beyond what prose alone can convey. The author's other work is listed under [More from the author](#more-from-the-author).

## When to use

Apply when **all** of the following hold:

- The project has stable concept-level structure (a matrix, ordered hierarchy, phase-to-skill binding, layered architecture, etc.)
- The structure does **not** change across `vX.Y.Z` releases
- You have empirical evidence that LLMs answer relationship questions poorly from prose alone
- The project already has `llms.txt` + `llms-full.txt` (the Answer.AI convention of Markdown summaries written for LLMs)

If your project is a single-purpose linear codebase or has internally churning structure, **don't** use this: `graph.jsonld` would become an editing burden.

## Install

```bash
git clone https://github.com/shimo4228/jsonld-knowledge-graph.git
mkdir -p ~/.claude/skills
cp -r jsonld-knowledge-graph/skills/jsonld-knowledge-graph ~/.claude/skills/jsonld-knowledge-graph
```

The skill itself is instructions. Its one script, the bundled lint (`scripts/graph_lint.py`), runs through [uv](https://docs.astral.sh/uv/) with the `pyld` JSON-LD library (`uv run --with pyld`, see [Verification](#verification)), so you need Python 3 and uv to lint a graph. Then ask Claude Code to design a `graph.jsonld` for the project, or type `/jsonld-knowledge-graph`.

## How it works

1. **When-to-use gate**: the skill checks four conditions (stable structure + release-stability + evidence that LLMs answer relationship questions poorly from prose + existing llms.txt and llms-full.txt) before any design work.
2. **9 reusable design moves**: for example, every node carries both a custom type and a schema.org type (dual `@type`), English and Japanese names live in one file with language tags, and versions and counts get no field at all. The full list is in [SKILL.md](skills/jsonld-knowledge-graph/SKILL.md) (in Japanese).
3. **Companion file wiring**: surfaces `graph.jsonld` to crawlers. The skill specifies what each companion file gains: `llms.txt` a note at the top telling AI readers to open `graph.jsonld` first, `llms-full.txt` a closing section headed by a question such as "How do X and Y relate as a graph?", and the README a short pointer at the end. The wording and placement belong to the companion skills: llms-txt-writer for the two llms files, readme-writer for the README pointer.
4. **Verification**: the bundled lint checks JSON validity, JSON-LD expansion, and the pitfalls that silently drop an edge.
5. **Maintenance contract**: explicit triggers for *when to edit* and *when NOT to edit* (routine releases, version bumps, ADR count changes). A multi-repo setup is one where a hub graph links several research lines (separate long-running projects, each in its own repository) and their supporting repositories. Edit triggers: a new stable concept in any project; in a multi-repo setup, also a supporting repository added or retired, and a new research line. A single-repo project skips the hub parts.
6. **Hugging Face mirror**: after a graph edit, the SKILL.md (in Japanese) suggests uploading `graph.jsonld` to a Hugging Face Datasets mirror. That step sends the file outside your machine and runs through a separate `hf-sync` skill, which this repository does not include; it is published in [claude-harness](https://github.com/shimo4228/claude-harness/tree/main/skills/hf-sync).

## Key concept: schema absence enforces invariants

The skill emphasizes that the strongest way to prevent a wrong relationship from being encoded is to **not define an edge type for it**. For example, if a project's research lines must remain siblings (never dependencies), the shared vocabulary defines `siblingOf` but deliberately does **not** define `dependsOn`. The vocabulary itself becomes a structural commitment. It holds under an explicit `@context` without `@vocab` (a catch-all that maps every undeclared key to a default namespace): there an undeclared `dependsOn` is dropped on expansion, and the lint reports it as DROPPED-KEY. Under `@vocab` the edge survives, and the lint has no check for an undeclared edge.

Similarly, by not putting `version` / `count` / `vX.Y.Z` field names in the schema, routine releases have no field to write volatile state into, under the same condition. This does not cover everything. Under `@vocab` an undeclared `version` key survives, and a value such as `"name": "X v2.1"` needs no field at all, so the lint's VOLATILE check and the grep under [Verification](#verification) are the backstop.

## What this skill does NOT do

| Concern | Use this instead |
|---|---|
| llms.txt / llms-full.txt prose design, the wording of their link lists, GEO (optimizing for AI search answers) | [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer) |
| How the README points to `graph.jsonld` and `llms.txt` | [readme-writer](https://github.com/shimo4228/readme-writer) |
| Project doc role overlap / freshness audit | [context-sync](https://github.com/shimo4228/context-sync) |
| File-level architecture maps (which file holds what, who calls whom) | Not kept in a file at all: derive them from the code when asked (Claude Code's LSP tool, an import graph such as `grimp`) |
| Article / blog post writing | [claude-skill-writing-ecosystem](https://github.com/shimo4228/claude-skill-writing-ecosystem) |

## Verification

After editing `graph.jsonld`, run the bundled lint (exit 0 = clean, 1 = findings, 2 = the file did not parse or expand):

```bash
uv run --with pyld python3 ~/.claude/skills/jsonld-knowledge-graph/scripts/graph_lint.py graph.jsonld

# Multi-repo setup: linting the hub graph with the research-line graphs also checks cross-file name drift (one IRI, different English names)
uv run --with pyld python3 ~/.claude/skills/jsonld-knowledge-graph/scripts/graph_lint.py hub/graph.jsonld line1/graph.jsonld
```

A clean graph prints `graph.jsonld: clean`. A graph whose explicit `@context` declares `siblingOf` but not `dependsOn`, with a node that uses both, prints:

```text
graph.jsonld: 1 finding(s)
  - DROPPED-KEY 'dependsOn' at https://example.org/line/a — not in @context and no @vocab: silently dropped on expansion
```

The lint's volatile-state check looks only at the keys `"version"`, `"versionNumber"`, `"adrCount"` and `"testCount"`. Version-shaped values need a separate look:

```bash
# Volatile state (should be empty)
grep -E '"version"|"versionNumber"|"adrCount"|v[0-9]+\.[0-9]+' graph.jsonld
```

Manual checks: [JSON-LD playground](https://json-ld.org/playground/), [schema.org validator](https://validator.schema.org/), and a relationship probe once crawlers have refreshed (the SKILL.md's rule of thumb is 1–2 weeks after the push): ask an LLM how two of the project's concepts relate, the same kind of question the When-to-use gate asks you to have evidence on. The SKILL.md (in Japanese) also checks whether the graph gets cited; treat citation as an unverified expectation, since the author's [Authorship Strategy ADR-0009](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0009-dual-entry-asymmetric-rebalance.md) (revised 2026-08-19) expects no near-term citation lift from the graph and keeps it for entity resolution.

## More from the author

- **[Banned from Wikidata Overnight](https://dev.to/shimo4228/banned-from-wikidata-overnight-i-believed-every-edit-was-compliant-but-all-109-items-were-57nl)** ([日本語](https://zenn.dev/shimo4228/articles/wikidata-ban-postmortem)): all 109 of the author's Wikidata items were deleted in one night, and every `sameAs` link (the statement that a node is the same thing as an outside record) to them in the author's graphs turned into a 404; it is why this skill points `sameAs` only at identifiers you control or have earned (ORCID, DOI, your own profiles).
- **[llms-txt-writer](https://github.com/shimo4228/llms-txt-writer)**: the companion skill for pages only AI reads (`llms.txt`, `llms-full.txt`, FAQ, glossary); it owns the wording of the reading-order note that this skill wires into `llms.txt` to send AI agents to `graph.jsonld` first.
- **[release-doi](https://github.com/shimo4228/release-doi)**: the release workflow for DOI-registered research repositories, whose concept DOIs this skill uses as the `@id` of each research line in a multi-repo setup.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with its own `graph.jsonld` linking the author's long-running projects, their DOIs and supporting repositories.

## License

MIT. See [LICENSE](LICENSE).

<details>
<summary>For tools and AI assistants</summary>

jsonld-knowledge-graph is an Agent Skill for Claude Code that designs and maintains a `graph.jsonld` file beside `llms.txt`, encoding a project's concepts and their relations as schema.org JSON-LD triples, for maintainers whose project has a matrix, hierarchy or phase structure that stays stable across releases. The graph holds the concept layer only; file-level structure is derived from the code when needed and never stored.

It exists because prose leaves relations implicit: an LLM asked how two parts of a project relate can answer poorly or wrongly from `llms.txt` and `llms-full.txt` alone, while triples state the relation outright and merge across files that reuse the same `@id`. The graph complements the prose files and does not replace them. Within the author's [Authorship Strategy](https://github.com/shimo4228/authorship-strategy) line, whose ADR-0006 asks a project to offer LLMs two entry points, this skill writes the concept-form graph and [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer) writes the prose-form navigator.

Canonical facts: MIT license; a `SKILL.md` (in Japanese), an `inspiration.md` with the origin and reference implementations, and one Python script, `skills/jsonld-knowledge-graph/scripts/graph_lint.py`, run through uv with `pyld`. Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (it never commits), so this repository can trail the harness between syncs. Requirements: Claude Code, where the skill is developed and tested (portable to other Agent Skills-compatible agents); Python 3 and uv for the lint; no API key. The SKILL.md also hands a Hugging Face Datasets mirror step to a separate `hf-sync` skill, which this repository does not include. Core rules: every node carries a custom type and a schema.org type; English and Japanese literals live in one file with `@language` tags; a forbidden relation is kept out by leaving its edge type out of the vocabulary, which holds only under an explicit `@context` without `@vocab` (there the lint reports the undeclared key as DROPPED-KEY; under `@vocab` the lint has no check for it); versions and counts have no field in the schema; research lines take their concept DOI as `@id`; `sameAs` points only at self-sovereign or earned identifiers, never Wikidata QIDs.

Example: `uv run --with pyld python3 ~/.claude/skills/jsonld-knowledge-graph/scripts/graph_lint.py graph.jsonld` prints `graph.jsonld: clean` and exits 0, or lists findings (DROPPED-KEY: a key the explicit `@context` does not map, silently dropped on expansion; URL-LITERAL: an IRI written as a string that became a literal, so the edge does not exist; VOLATILE: a version or count key) and exits 1; given several graphs it also reports NAME-DRIFT, one IRI carrying different English names across files.

Links: [skills/jsonld-knowledge-graph/SKILL.md](skills/jsonld-knowledge-graph/SKILL.md) is the skill itself; [inspiration.md](skills/jsonld-knowledge-graph/inspiration.md) names the reference implementations, among them the hub's [graph.jsonld](https://github.com/shimo4228/shimo4228/blob/main/graph.jsonld); [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are this repository's machine-readable summary and reference. The parent line is [Authorship Strategy](https://github.com/shimo4228/authorship-strategy), concept DOI [10.5281/zenodo.20263316](https://doi.org/10.5281/zenodo.20263316).

</details>
