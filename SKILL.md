---
name: entity-territory-audit
description: Audit the named-entity territory of a target URL against up to 5 well-ranked competitor pages on the same topic, in FR or EN. Extract every meaningful entity from each page (tools, concepts, methodologies, persons, brands, standards, technologies, metrics, etc.), type it, normalize aliases, then produce a typed coverage matrix and a priority-scored list of entities the target page is missing. Fully autonomous - performs web search, fetching and entity extraction by reasoning, without relying on any external NER, MCP server or third-party SEO tool. Trigger for requests such as "cartographie le territoire entitaire de cette URL", "fais un audit d'entites sur ce sujet", "quelles entites me manquent par rapport au top Google", "entity gap audit", "trouve la dette entitaire de ma page", or "compare les entites nommees de ma page vs concurrents".
---

# Entity Territory Audit

Use this skill to map the **named-entity territory** of a topic and measure the entity gap between a target URL and up to 5 well-ranked competing pages. The unit of analysis is the **typed entity**, not the keyword and not the subtopic. Output is a typed coverage matrix plus a priority-scored list of entities the target page should add.

This skill is **autonomous by design**: it does not call MCP servers, third-party SEO tools, or external NER pipelines. Entity extraction is performed by direct LLM reasoning over the live page text. Web search and fetching go through the standard `WebSearch` and `WebFetch` tools.

This skill is **distinct from** `semantic-gap-blindspot-auditor`. That one mixes terms, entities, subtopics, and blindspots into a broad editorial gap audit. This one focuses **only** on named entities, with strict typing, alias normalization, and a defensible priority score. Run them in sequence when a deep audit is needed; do not merge their outputs.

## Required Inputs

- `target_url` - the page to audit
- `topic` - the thematic angle or main keyword the audit must center on (FR or EN)

## Optional Inputs

- `competitor_urls` - explicit list of up to 5 URLs. When provided, use exactly these and skip discovery.
- `max_competitors` - default `5`, hard cap at `5`.
- `language` - default auto-detected from `target_url`. Supported: `fr`, `en`.
- `output_mode` - `inline` (default, render the report directly in chat) or `file` (write `entity-audit-<slug>-<YYYY-MM-DD>.md` in the current working directory).
- `entity_cap` - default `60` final entities in the universe. Trim noise below the cap if needed.

## Core Rules

- Work only from **live page text**. Never extract entities from memory, training data, or assumed knowledge of the page.
- An entity is a **named, referable thing** with a typeable identity (tool, concept, person, standard, etc.). Generic descriptors and pure adjectives are not entities.
- Every entity must be **typed**. Use the open taxonomy in `references/entity-types.md`. When uncertain, use `Other` with a one-word subtype, never leave a type blank.
- Normalize **aliases** explicitly. `GSC` and `Google Search Console` are the same entity. Track all variants found, but report the canonical name once. Apply the alias rules in `references/entity-types.md`.
- Implicit coverage counts. If the target page describes the concept in plain words without naming it, mark it `implicit_present`, not `absent`.
- Never recommend an entity that does not fit the page's primary intent. Topical fit beats SERP frequency.
- Quote evidence. Every claim that an entity appears or is missing must be supported by a short verbatim snippet (1-2 sentences max).
- The competitor set is fixed at the start of the audit. Do not silently swap pages mid-run.
- File deletion is forbidden without explicit user permission. This skill never deletes anything.

Read [entity-types.md](./references/entity-types.md) before extraction. Read [scoring-model.md](./references/scoring-model.md) before prioritization. Read [report-template.md](./references/report-template.md) before producing the final output.

## Workflow

### 1. Frame the audit

- Confirm `target_url`, `topic`, language, and the competitor source (auto-discovery or user-provided list).
- Detect language from the target URL's `<html lang>` or visible text. Lock it for the rest of the run. FR target -> FR competitors. EN target -> EN competitors. Do not mix unless the user explicitly asks.
- State the framing in 2-3 lines before doing any extraction so the user can correct course early.

### 2. Identify competitors (skip if `competitor_urls` is provided)

- Run `WebSearch` on `topic`, language-localized when possible (e.g. add `site:.fr` or French-language operators when language is `fr`).
- From the results, select up to 5 URLs that satisfy at least three of these signals:
  - high apparent organic ranking on the topic
  - editorial depth (long-form content, real expertise, not pure listicles)
  - independent domain (avoid taking 5 pages from the same site)
  - direct topical match (do not include adjacent topics)
  - publication recency or maintained content (avoid clearly stale pages)
- Exclude paywalled URLs you cannot fetch, social media threads, video pages without transcript, and the target's own domain.
- Print the final competitor set with one-line justification per URL before fetching. The user can override at this point.

### 3. Fetch all pages

- Use `WebFetch` on the target URL and on each competitor URL.
- For each page, capture the visible main content. Strip nav, footer, related-article rails, and comments. Keep `H1`, `H2`, `H3`, intro, body, lists, FAQ, and image alt text when relevant.
- If a fetch fails or returns thin content, retry once with a stricter prompt asking for the article body. If still empty, drop that competitor and continue with the rest. Never silently substitute a different URL without telling the user.
- Stop and ask the user for a paste or alternative if the **target** page cannot be fetched. The audit is meaningless without the target text.

### 4. Extract entities per page

For each page (target + each competitor), produce a JSON-shaped block in working memory:

```
{
  "url": "...",
  "is_target": true|false,
  "rank": 1..5 | null,
  "language": "fr"|"en",
  "entities": [
    {
      "canonical": "Google Search Console",
      "aliases": ["GSC", "Search Console"],
      "type": "Tool",
      "subtype": "SEO analytics",
      "mentions": 4,
      "evidence": "Une page sur l'audit SEO doit mentionner explicitement Google Search Console..."
    },
    ...
  ]
}
```

Extraction rules:

- Read the entity-types taxonomy in `references/entity-types.md` first and stay inside it.
- Pull only entities that are **explicitly named** in the page text. Do not infer entities from context.
- For each entity, count `mentions` as the number of distinct surface occurrences (case- and accent-insensitive after normalization).
- Capture one short `evidence` snippet per entity, taken verbatim from the page.
- When two surface forms refer to the same thing, pick the longest unambiguous form as `canonical` and put the rest in `aliases`. Do not merge across **different** entities even if names look similar (e.g. `GA4` and `Universal Analytics` are different entities).
- Detect transliteration and language variants in mixed FR/EN content (e.g. `Search Console` vs `console de recherche` -> same entity if context confirms).
- Skip generic role nouns (`an SEO consultant`, `the user`), pure adjectives (`responsive`, `mobile-friendly`) and stop-style phrases.

### 5. Build the entity universe and coverage matrix

- Merge all per-page entity lists into a single universe, deduping by canonical name plus alias overlap.
- For each entity in the universe, compute:
  - `target_mentions` (0 if absent)
  - `target_status`: `present` (>=1 explicit mention), `implicit_present` (concept covered without naming), `absent`
  - `competitor_coverage`: count of competitors (0..5) where the entity appears explicitly at least once
  - `competitor_positions`: list of ranks where the entity appears (e.g. `[1, 2, 4]`)
  - `total_competitor_mentions`: sum across competitors
  - `type` and `subtype`: agreed canonical type. If different pages typed the same entity differently, pick the type used by the majority and note the disagreement in a comment.
- Cap the universe at `entity_cap` (default 60). Drop entities with `competitor_coverage <= 1` and `target_mentions == 0` first - these are noise.

### 6. Score and prioritize

- Apply the formula in `references/scoring-model.md` to every entity in the universe.
- Sort by descending priority score.
- Bucket entities into:
  - `must add` (score 75-100)
  - `strong opportunity` (50-74)
  - `nice to add` (25-49)
  - `already covered` (target presents the entity well)
  - `skip` (under 25, kept only as appendix)
- For each `must add` and `strong opportunity` entity, derive a one-line **insertion guidance**: where in the target page it should appear (existing H2, new section, FAQ, comparison block, definition, internal link target) and **how** to integrate it (definition, mention with link, dedicated paragraph, table row, etc.).

### 7. Verify before finalizing

- Spot-check at least 3 high-priority entities: re-read the target page text to confirm `target_status` is correct.
- For any entity claimed `absent across all competitors`, re-check at least one competitor body to avoid false negatives caused by partial fetch.
- Confirm no recommended entity is off-intent for the target page.

### 8. Produce the report

- Build the markdown report using the structure in `references/report-template.md`.
- If `output_mode` is `inline` (default), render the full report directly in the chat reply.
- If `output_mode` is `file`, write the report to `entity-audit-<slug-of-target-url>-<YYYY-MM-DD>.md` in the current working directory and confirm the path. Do **not** delete or overwrite a same-name file without asking.
- Always include the competitor list with rationale, the typed coverage matrix, the priority groups, and per-entity detail cards for the top recommendations.

## Output Contract

When the user asks for an audit, return at minimum:

1. **Audit framing**: target URL, topic, detected language, competitor URLs with ranks, total entities in universe, coverage stats (target coverage rate vs competitor average).
2. **Coverage matrix**: one row per entity in the universe (capped), columns = `Entity | Type | Target | C1 | C2 | C3 | C4 | C5 | Cov | Score | Bucket`.
3. **Priority groups**: lists of entities under `Must add`, `Strong opportunity`, `Nice to add`, `Already covered`. Each entity shown with type, score, and one-line evidence.
4. **Top entity detail cards**: for the top 10 priority entries, expand into: canonical name, type/subtype, aliases found, score breakdown, where the target stands today, where and how to integrate it, one verbatim evidence snippet per side.
5. **Appendix**: skipped entities and noise (compact list), and a short note on disagreements in typing if any.
6. **Methodology footer**: language, fetch outcomes (which URLs were retried/dropped), and acknowledged limits (e.g. one competitor failed to fetch, results capped to 4).

## Guardrails

- Do not output entities that are not actually in the page text.
- Do not collapse two distinct entities into one because their names look similar.
- Do not split one entity into two because the page used different surface forms - that is what aliases are for.
- Do not score an entity high just because it appears across competitors if it is off-topic for the target page.
- Do not propose a rewrite. Only recommend additions, expansions, or local edits sufficient to introduce the entity naturally.
- Do not silently downgrade `competitor_coverage` to fit a story - if 5/5 competitors mention an entity that the target lacks, that finding stays.
- Do not reuse stale data between runs. Each audit must re-fetch.
- Never delete the produced report file (or any file) without explicit user permission.

## Quick Triggers

These user requests should trigger the skill:

- "Cartographie le territoire entitaire de cette URL sur ce sujet."
- "Fais un audit des entites de ma page vs les meilleurs concurrents."
- "Quelles entites me manquent par rapport au top Google sur ce theme ?"
- "Trouve la dette entitaire de cette page."
- "Compare les entites nommees de ma page aux 5 meilleures sur cette requete."
- "Run an entity gap audit on this URL for this topic."
- "List the typed entities I am missing vs the SERP top 5."
