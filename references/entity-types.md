# Entity Types and Alias Rules

Read this file before extracting entities. The taxonomy is open at the `subtype` level but **closed at the `type` level**: every entity must fall into one of the categories below.

## 1. What counts as an entity

An entity is a **named, referable thing** with a typeable identity. It must satisfy all of:

1. It has a proper name, an acronym, or a stable technical label that other authors would recognize.
2. It can be linked to a Wikipedia page, a product page, a doc page, an organization page, a standards body, or a clearly identifiable definition.
3. It is referenced in the page text, not assumed from context.
4. It carries semantic weight: removing it from the page would change the meaning of a section, not just the wording.

**Counterexamples** (not entities):

- generic descriptors: `responsive`, `scalable`, `mobile-friendly`, `optimal`
- role nouns without proper name: `the consultant`, `our team`, `users`, `the reader`
- vague phrases: `the right approach`, `best practices`, `modern frameworks`
- pure stop topics: `things to consider`, `how it works`

## 2. Closed type vocabulary

Use exactly one of these `type` values. Add any clarifier as a free-text `subtype`.

- `Tool` - software, app, plugin, extension, CLI, library, SaaS product, API. Subtype examples: `SEO analytics`, `LLM`, `IDE`, `MCP server`.
- `Concept` - abstract idea, model, theory, paradigm. Subtype examples: `Information Retrieval`, `Learning Theory`, `UX principle`.
- `Method` - methodology, framework, process, workflow, recipe, audit type. Subtype examples: `audit`, `algorithm`, `protocol`, `playbook`.
- `Standard` - norm, regulation, specification, format, protocol, schema. Subtype examples: `IETF RFC`, `W3C spec`, `ISO`, `legal regulation`.
- `Technology` - underlying technology, technique, architecture, paradigm-as-tech. Subtype examples: `embeddings`, `BERT`, `RAG`, `transformer`.
- `Person` - identified human (author, founder, researcher, public figure).
- `Organization` - company, institution, lab, working group, foundation, agency.
- `Brand` - branded product line, service, or commercial name not better described as `Tool` or `Organization`.
- `Metric` - named indicator, score, KPI, benchmark, evaluation. Subtype examples: `Core Web Vital`, `SEO score`, `dataset benchmark`.
- `Event` - conference, milestone, launch, release, public event with a recognizable name.
- `Place` - country, region, city, market when materially relevant to the topic.
- `Other` - last resort. Always fill `subtype` with one descriptive word. Use sparingly. If `Other` exceeds 10% of the universe, the typing is too lazy - retype.

## 3. Required entity record shape

Every extracted entity must produce this record:

```
{
  "canonical": "<longest unambiguous form>",
  "aliases": ["<other surface forms found in the text>"],
  "type": "<one of the closed vocabulary>",
  "subtype": "<short free-text>",
  "mentions": <int>,
  "evidence": "<verbatim snippet 1-2 sentences containing one occurrence>"
}
```

## 4. Alias normalization rules

Apply these rules **before** computing the universe. The goal is to count `GSC` and `Google Search Console` as the same entity.

### 4.1 Merge as the same entity

- acronym and full form: `GSC` <-> `Google Search Console`, `RAG` <-> `Retrieval-Augmented Generation`
- short form and full form, when context confirms identity: `Search Console` (in a Google context) -> `Google Search Console`
- inflection and plural: `embeddings` and `embedding` -> `embeddings`
- accent variants: `referencement` <-> `referencement`
- spacing and punctuation variants: `text-embedding-3-small` <-> `text embedding 3 small`
- localized translation when context is unambiguous: `Recherche Google` <-> `Google Search` (only when both refer to the product, not the verb)
- versioned naming when the page treats them as one: `GPT-5` and `GPT-5 mini` -> only merge if the page does not treat them as distinct products. By default, **keep them separate**.

### 4.2 Do not merge

- different versions when the page treats them as distinct: `GA4` vs `Universal Analytics`, `Python 2` vs `Python 3`
- different products of the same vendor: `Search Console` vs `Google Analytics`, `Vertex AI` vs `Gemini API`
- a method and a tool with the same name: `Pagerank` (algorithm) vs `Pagerank` (a hypothetical product) - check context, type accordingly, do not merge.
- homonyms: `Apache` (foundation) vs `Apache HTTP Server` vs `Apache Spark` - keep separate, type each one.

### 4.3 Choosing the canonical form

- pick the **longest unambiguous form** as canonical (`Google Search Console`, not `GSC`)
- if the page consistently uses an acronym and never spells out the full form, keep the acronym as canonical and add the spelled-out form to `aliases` (when known with high confidence)
- never invent an alias that is not in the page text

## 5. Cross-page reconciliation

When merging per-page lists into the universe:

- two entities from different pages merge if their `canonical` matches **or** if their `canonical` and any `alias` overlap unambiguously
- when types disagree across pages for the same entity, use the type chosen by the **majority** of pages, with the longest evidence; record the disagreement in the audit appendix
- mention counts are page-local. The universe-level coverage signal is `competitor_coverage` (count of distinct pages, not summed mentions)

## 6. Implicit coverage

When checking the **target page** specifically, distinguish three states:

- `present`: the entity is named at least once explicitly (counts as covered)
- `implicit_present`: the concept is described in the body but the entity is never named (still considered partially covered; lower target gap weight)
- `absent`: neither named nor described

Implicit coverage requires a verbatim snippet showing the description. Never declare `implicit_present` from inference alone.

## 7. Anti-patterns to refuse

- making up an entity because "the page should mention it"
- splitting one entity into two because the page used different surface forms (use `aliases`)
- merging two entities because their names are similar (`GA4` and `Google Ads` are not the same)
- typing as `Concept` to avoid choosing - if it is software, type it `Tool`; if it is a benchmark, type it `Metric`
- letting `Other` become a dumping ground - retype if usage is high
