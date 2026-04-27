# entity-territory-audit

> Agent Skill for **typed entity-gap audits** — compare a target URL to up to 5 well-ranked competitors on the same topic, and surface the named entities the page is missing, scored by priority.

Works natively as an Agent Skill in **Claude Code**, **OpenAI Codex CLI**, **Claude.ai / Claude Desktop**, and as a system-prompt drop-in for **any LLM agent** (Cursor, Aider, ChatGPT custom instructions, etc.).

---

## What it does

Standard semantic SEO audits mix lexical density, subtopic coverage, and tools-driven scores. This skill does only one thing, and does it precisely:

It builds a **typed entity universe** from the target URL plus up to 5 competitor URLs, normalizes aliases (`GSC` ⟷ `Google Search Console`), and produces a **priority-scored list of entities the target page should add**, with editorial integration guidance.

The unit of analysis is the **named entity**, not the keyword and not the subtopic. Every entity is typed (`Tool`, `Concept`, `Method`, `Standard`, `Technology`, `Person`, `Organization`, `Brand`, `Metric`, `Event`, `Place`, `Other`) and scored from 0 to 100 across five signals.

---

## How it works

1. **Frame** the topic and detect language (FR or EN).
2. **Discover** up to 5 competing URLs via web search, or use a list you provide.
3. **Fetch** the live text of every page (target + competitors).
4. **Extract** entities by direct LLM reasoning — no external NER, no MCP server, no third-party SEO tool required.
5. **Normalize aliases**, build the universe, compute coverage matrix.
6. **Score and bucket** entities into `must add` / `strong opportunity` / `nice to add` / `already covered`.
7. **Output** a markdown report with matrix, priority groups, top-10 detail cards, and editorial integration plan.

---

## What you get

- Coverage matrix: `Entity | Type | Target | C1..C5 | Cov | Score | Bucket`
- Priority groups with one-line rationale
- Top-10 entity cards with insertion guidance (where + how on the page)
- Verbatim evidence on both sides (competitor + target)
- Methodology footer (fetch outcomes, type-criticality overrides, known limits)

The full output contract is documented in [`references/report-template.md`](./references/report-template.md). Scoring formula in [`references/scoring-model.md`](./references/scoring-model.md). Type taxonomy + alias rules in [`references/entity-types.md`](./references/entity-types.md).

---

## Installation

The skill is the same file structure everywhere. Different agents look for it in different folders.

### Claude Code (CLI / Desktop)

User-level (available in every project):

```bash
git clone https://github.com/JuJu78/entity-territory-audit.git \
  ~/.claude/skills/entity-territory-audit
```

Or project-level (only this repo):

```bash
git clone https://github.com/JuJu78/entity-territory-audit.git \
  .claude/skills/entity-territory-audit
```

Restart Claude Code if it doesn't pick up the new skill.

### OpenAI Codex CLI

Codex auto-discovers skills from `.agents/skills/` at user, project, or system level.

User-level:

```bash
git clone https://github.com/JuJu78/entity-territory-audit.git \
  ~/.agents/skills/entity-territory-audit
```

Project-level:

```bash
git clone https://github.com/JuJu78/entity-territory-audit.git \
  .agents/skills/entity-territory-audit
```

Then invoke explicitly with `$entity-territory-audit` or `/skills` in your Codex prompt, or let Codex pick it up implicitly when your request matches the skill description.

> See [Codex's official Skills docs](https://developers.openai.com/codex/skills) for management commands.

### Claude.ai / Claude Desktop (Agent Skills)

If Agent Skills are enabled in your workspace:

1. Zip the skill folder: `zip -r entity-territory-audit.zip entity-territory-audit/`
2. In your Claude.ai workspace, go to **Skills** → **Upload skill**
3. Upload the zip; the skill becomes available in any conversation in that workspace

If your plan does not include Agent Skills, paste the contents of `SKILL.md` into a Claude **Project**'s custom instructions, and reference the `references/*.md` files when needed.

### Cursor / Aider / generic LLM agents

These agents have no native skill system, but the format works as content:

- **Cursor**: paste `SKILL.md` content into your `.cursor/rules` file (or `.cursorrules` legacy)
- **Aider**: pass `SKILL.md` to `aider --read SKILL.md`
- **ChatGPT custom GPT / Claude Project**: paste `SKILL.md` as the system prompt, attach `references/*.md` as knowledge files
- **Any API agent**: prepend `SKILL.md` to your system prompt; load references on demand when the workflow refers to them

---

## Quick start

Once installed, invoke the skill with any of these (FR or EN):

- *Cartographie le territoire entitaire de https://example.com sur le sujet "audit technique SEO"*
- *Fais un audit d'entités sur cette URL vs les 5 meilleurs concurrents*
- *Trouve la dette entitaire de cette page sur ce thème*
- *Run an entity gap audit on this URL for the topic "headless CMS comparison"*
- *List the typed entities I am missing vs the SERP top 5*

You can also pass an explicit competitor list to skip web search:

```
target_url: https://example.com/post
topic: "audit technique SEO"
competitor_urls:
  - https://competitor-a.fr/article
  - https://competitor-b.com/guide
```

---

## License

[MIT](./LICENSE) — use it, fork it, ship it. A credit link back is appreciated but not required.

---

## Credit

Designed by [@JuJu78](https://github.com/JuJu78) ([julien-gourdon.fr](https://julien-gourdon.fr)) with [Claude](https://claude.com/claude-code).
