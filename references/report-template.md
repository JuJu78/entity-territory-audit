# Report Template

Use this exact structure for the final entity audit report. Skip a section only if its content is empty after the run, and say so explicitly.

The report can be rendered inline in the chat or written to a file. The structure is identical in both modes.

## File naming (when `output_mode == "file"`)

`entity-audit-<slug>-<YYYY-MM-DD>.md`

- `slug`: lowercase, hyphenated derivation of the target URL path (e.g. `journal-seo-ia-audit-technique`)
- `YYYY-MM-DD`: today's date

Do not overwrite an existing file with the same name without asking. Suggest a `-v2` suffix instead.

## Skeleton

```markdown
# Entity Territory Audit - <topic>

**Target URL**: <target_url>
**Topic**: <topic>
**Language**: <fr|en>
**Date**: <YYYY-MM-DD>
**Competitors fetched**: <N>/5

## 1. Audit framing

- Detected language: <fr|en>
- Topic intent (one line): <...>
- Competitor selection method: <auto-discovery via WebSearch | user-provided list>
- Type criticality override applied: <none | "Tool +2" | ...>
- Universe size after capping: <K> entities (cap = <entity_cap>)
- Coverage stats:
  - Target explicit coverage: <X> / <K> (<percent>%)
  - Target implicit coverage: <Y> / <K> (<percent>%)
  - Average competitor coverage: <Z> / <K> (<percent>%)
  - Entity debt (must-add count): <M>

## 2. Competitor set

| Rank | URL | Why selected |
|---|---|---|
| 1 | <url> | <one line: authority, topical match, depth signal> |
| 2 | ... | ... |

If a competitor was dropped after a failed fetch, list it under a `Dropped` subheading with the reason.

## 3. Coverage matrix

Sort by descending priority score.

| # | Entity | Type | Subtype | Target | C1 | C2 | C3 | C4 | C5 | Cov | Score | Bucket |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Google Search Console | Tool | SEO analytics | absent | 4 | 3 | 2 | 5 | 3 | 5/5 | 97 | must add |
| 2 | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |

Cell convention:

- `Target` column: `present (n)`, `implicit`, or `absent`
- `Ci` columns: integer mention count, or empty cell
- `Cov`: `competitor_coverage / N`

## 4. Priority groups

### 4.1 Must add (score 75-100)

For each entry, one bullet:

- **<Canonical>** _(Type / Subtype, score N)_ - <one-line evidence: why competitors carry it, why target lacks it>

### 4.2 Strong opportunity (50-74)

Same format.

### 4.3 Nice to add (25-49)

Same format. Keep the list short.

### 4.4 Already covered

List of entities the target already presents well (>= 3 explicit mentions). One-liner each.

## 5. Top entity detail cards

Render up to 10 cards from the top of the priority list. Use this exact card shape:

---

### <Canonical name>

- **Type / Subtype**: <Type> / <subtype>
- **Aliases observed**: <alias1>, <alias2>
- **Score**: <total> = coverage <p1> + rank <p2> + gap <p3> + type <p4> + density <p5>
- **Target status today**: <absent | implicit | present (n mentions)>
- **Competitor presence**: <count>/<N>, ranks <[1, 2, 3]>, total mentions <m>
- **Why it matters here**: <one-line topical justification, not generic>
- **Where to integrate on the target page**: <existing H2 name | new section title | FAQ pair | comparison block | definition list | internal link target>
- **How to integrate**: <definition + 1 sentence | mention with link to canonical source | dedicated paragraph + example | row in comparison table | etc.>
- **Evidence (competitor verbatim)**: "<short quote from one competitor page>" - <competitor URL or rank>
- **Evidence (target side)**: <"<short quote of the implicit description>" if implicit, or "Not found in body, H2 or FAQ" if absent>

---

## 6. Editorial integration plan

A 5- to 8-line plan ordering the most impactful additions:

1. <First entity to add, where, in what form>
2. <Second>
3. ...

Prefer additions that share a section: if three entities all belong in the same H2 expansion, group them.

## 7. Appendix

### 7.1 Skipped entities (score < 25)

Compact list: `<name> (Type, score)` per line. No justification needed.

### 7.2 Typing disagreements

Only when the same entity was typed differently across pages. One line per case:

- `<entity>`: <type chosen> (used by Cx, Cy); rejected: <other type> (used by Cz)

### 7.3 Methodology footer

- Fetch outcomes: <"all 5 fetched ok" | "C3 dropped: 403 after retry">
- Type criticality override: <yes/no, which type, why>
- Cap applied: <yes/no>
- Sanity check spot-count: <number of must-add entities re-verified>
- Known limits: <e.g. "C2 is paywalled, only the visible intro was used">
```

## Tone and length guidance

- Prefer evidence over adjectives. Every claim carries a verbatim snippet.
- Keep card descriptions tight. The decision points are: where on the page, what shape of edit, what one quote proves it.
- Do not pad the priority lists. If `must add` is empty, write `none` and explain why (e.g. target already strong on this topic).
- Do not invent integration suggestions that contradict the page's intent. When a top-scored entity does not fit, downgrade it before publishing.
