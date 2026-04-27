# Entity Priority Scoring Model

Read this file when scoring entities for the priority list. The score is a deterministic 0-100 number computed from five signals. The same inputs must always yield the same score.

## 1. Score formula

`priority_score = competitor_coverage_pts + competitor_rank_weight_pts + target_gap_pts + type_criticality_pts + mention_density_pts`

Maximum is 100. Each component is bounded.

| Component | Max | What it measures |
|---|---|---|
| Competitor coverage | 30 | How many of the N competitors mention the entity |
| Competitor rank weight | 25 | Whether top-ranked competitors mention it |
| Target gap | 25 | How absent the entity is on the target page |
| Type criticality | 10 | Heavier weight for types that materially shape topical authority |
| Mention density | 10 | Total mentions across competitors, normalized |

`N` is the actual number of fetched competitors (1..5). All ratios are computed against `N`, not the requested cap.

## 2. Component scoring

### 2.1 Competitor coverage (0-30)

`competitor_coverage_pts = round(30 * (competitor_count / N))`

| competitor_count / N | Points |
|---|---|
| 5/5 | 30 |
| 4/5 | 24 |
| 3/5 | 18 |
| 2/5 | 12 |
| 1/5 | 6 |
| 0/5 | 0 |

### 2.2 Competitor rank weight (0-25)

For each competitor `i` (rank 1 = best), if the entity appears, add `weight_i`:

| Rank `i` | `weight_i` |
|---|---|
| 1 | 10 |
| 2 | 7 |
| 3 | 5 |
| 4 | 2 |
| 5 | 1 |

Cap the sum at 25. Skipped ranks (no competitor at that position) contribute 0.

### 2.3 Target gap (0-25)

| Target status | Points |
|---|---|
| `absent` | 25 |
| `implicit_present` | 14 |
| `present`, mentioned 1 time | 7 |
| `present`, mentioned 2 times | 3 |
| `present`, mentioned >= 3 times | 0 |

Multiply target mentions by ratio of explicit-form mentions vs alias mentions only when relevant; otherwise just count distinct surface occurrences.

### 2.4 Type criticality (0-10)

Default weights. Override only when the topic justifies a different emphasis (e.g. a topic about regulation lifts `Standard` weight).

| Type | Default points |
|---|---|
| Tool | 9 |
| Standard | 9 |
| Method | 8 |
| Technology | 8 |
| Concept | 7 |
| Metric | 7 |
| Brand | 6 |
| Organization | 5 |
| Person | 4 |
| Event | 3 |
| Place | 2 |
| Other | 3 |

If the topic is overtly about a type (e.g. "best SEO tools"), boost that type by `+2`, capped at 10. Apply this override at the audit start, not per entity, and disclose it in the methodology footer.

### 2.5 Mention density (0-10)

`mention_density_pts = min(10, round(2 * log2(1 + total_competitor_mentions)))`

| `total_competitor_mentions` | Points |
|---|---|
| 0 | 0 |
| 1 | 2 |
| 3 | 4 |
| 7 | 6 |
| 15 | 8 |
| >= 31 | 10 |

This component punishes one-off namedrops and rewards entities that competitors return to repeatedly.

## 3. Buckets

Map the final score to a bucket:

| Score | Bucket | Meaning |
|---|---|---|
| 75 - 100 | `must add` | High coverage by strong competitors AND missing on target. Address first. |
| 50 - 74 | `strong opportunity` | Clear gap or strong corroboration; recommend integration |
| 25 - 49 | `nice to add` | Moderate signal; integrate only if natural |
| 0 - 24 | `skip` | Noise or already covered. Keep as appendix only. |

**Override**: when `target_status == present` with `>= 3 mentions`, the entity goes to `already covered` regardless of score. Do not recommend more mentions of an entity already named multiple times.

## 4. Tie-breakers

When two entities share the same score, prioritize in this order:

1. higher `competitor_rank_weight_pts` (top-ranked corroboration matters more than sheer count)
2. higher `target_gap_pts` (favor what is actually missing)
3. higher `type_criticality_pts`
4. alphabetical by canonical name (deterministic)

## 5. Sanity checks before publishing the score

For every entity in `must add` and `strong opportunity`, verify:

1. Is this entity **actually present** in the listed competitor pages? Re-check at least one verbatim evidence snippet.
2. Is the **target status** correct? Re-search target text for the canonical name and each alias.
3. Is the entity **on-intent** for the target page's primary purpose? If integrating it would force the page off its core intent, downgrade to `nice to add` or `skip`.
4. Is the `type` the right one? Wrong typing distorts criticality and the editorial integration suggestion.

If any check fails, fix the data and recompute. Do not publish a `must add` recommendation that fails a sanity check.

## 6. Worked example

Topic: "audit technique SEO". Target absent. Competitors fetched: 5.

Entity: `Google Search Console`

- competitor_count = 5/5 -> 30 pts
- ranks where present: 1, 2, 3, 4, 5 -> 10 + 7 + 5 + 2 + 1 = 25 pts (capped at 25)
- target_status = `absent` -> 25 pts
- type = `Tool`, topic about audit (not specifically tools) -> 9 pts
- total_competitor_mentions = 17 -> 8 pts

`priority_score = 30 + 25 + 25 + 9 + 8 = 97` -> bucket `must add`.

Entity: `John Mueller`

- competitor_count = 1/5 -> 6 pts
- ranks where present: 4 -> 2 pts
- target_status = `absent` -> 25 pts
- type = `Person` -> 4 pts
- total_competitor_mentions = 1 -> 2 pts

`priority_score = 6 + 2 + 25 + 4 + 2 = 39` -> bucket `nice to add`.

The formula correctly downweights the lone Person mention and elevates the cross-corroborated Tool.
