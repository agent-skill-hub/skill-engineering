---
name: skill-engineering
description: "Use when creating or restructuring any skill that involves domain knowledge — deciding what stays inline vs sinks to CSV/RAG, or when an existing SKILL.md exceeds 300 lines and needs restructuring."
---

# Skill Engineering - Knowledge-Base Skill Design

## Trigger Conditions

- User asks to create a new skill that involves domain knowledge
- User asks to restructure an existing skill
- A SKILL.md exceeds 300 lines and most content is data (not rules)

## Iron Rule: Do Not Over-Engineer

If total knowledge entries <= 50, DO NOT create CSV, scripts, or external data files. Inline everything in SKILL.md. Most skills are Light level — keep them that way.

## Step 0: Domain Scope — Single or Multi?

Before classifying knowledge by form and scale, first determine **domain scope**:

| Signal | Scope | Architecture |
|--------|-------|-------------|
| All knowledge shares the same trigger conditions and usage scenarios | **Single domain** | 1 skill — proceed to Knowledge Classification below |
| Knowledge splits into 2+ groups with **independent triggers, independent process descriptions, and independent usage scenarios** | **Multi-domain** | Retrieval layer + domain skills (see below) |

**"Independent" means:** a user working in domain A rarely needs domain B's process descriptions in the same context. If they do, it's for looking up a term or config — not reading the full flow.

### Multi-Domain Architecture

Each box below is an **independent skill** (its own directory under `skills/`, its own SKILL.md, its own trigger conditions). They are peers, not nested:

```
skills/
  retrieval-skill/          ← Shared search layer
    SKILL.md                ← Search guide only (which domain, what command)
    data/*.csv              ← All entity data (terminology, configs, etc.)
    scripts/                ← BM25 search engine

  domain-a/                 ← Independent skill, triggered by domain A tasks
    SKILL.md                ← Trigger + rules + flow chains (<=500 lines)
    references/             ← Long-form details for this domain only

  domain-b/                 ← Independent skill, triggered by domain B tasks
    SKILL.md
    references/
```

Retrieval skill is triggered by entity lookups (term, config, queue name). Domain skills are triggered by their specific task context (e.g., "debug payment flow" triggers domain-a, not domain-b). Multiple skills can be loaded in the same session when needed.

**Separation of concerns:**
- **Retrieval skill** owns entity data (countable: terms, configs, queues, services). No process descriptions. SKILL.md is just a search guide.
- **Domain skills** own process descriptions (flows, architectures, auth chains). Entity lookups go through the retrieval skill's BM25 search — no need to duplicate terminology tables.
- Each domain skill's SKILL.md stays within the 500-line budget because it only covers one domain.

**When NOT to split into multi-domain:**
- Domains are tightly coupled (understanding A requires reading B's full flow) → single skill
- Total process descriptions across all domains < 100 lines → single skill, inline everything
- Only 1-2 domains → single skill with references is fine

### How to Identify Domains

A domain = a group of knowledge with a shared trigger condition. Test: "If a user asks about X, do they need Y's full flow description in the same context?"

- If yes → same domain
- If they only need to look up a term/config from Y → separate domains, connected by retrieval layer

## Knowledge Classification → Action

**Applies within a single domain (or a single-domain skill).** For multi-domain setups, apply this classification independently within each domain skill and the retrieval skill.

Knowledge has two orthogonal dimensions: **form** (what it looks like) and **scale** (how many).

### Dimension 1: Form — Is it a countable entity or a process description?

| Form | Examples | Key Test |
|---|---|---|
| **Countable entity** | A term, a config, an error code, a queue, a service | Can you put it in one CSV row? → Entity |
| **Process description** | A multi-step flow, an architecture diagram, a processing chain, an auth sequence | Is it narrative steps/explanations that lose meaning when flattened? → Process |

### Dimension 2: Scale — How many?

**For countable entities:**

| Entry Count | Schema Fixed? | Action |
|---|---|---|
| <=20 | — | Inline as rules/principles in SKILL.md |
| 20-50 | — | Terminology table in SKILL.md |
| >50 | Yes | `data/*.csv` + BM25 search script |
| >50 | No, but flat | Flatten to fixed schema → CSV + BM25 |
| >50 | No, has relationships | External RAG / MCP |

How to count entries: each distinct knowledge entity (a config, a rule, a term, an error code) = 1 entry.

**For process descriptions:**

| Total Lines | Action |
|---|---|
| <=30 lines | Inline in SKILL.md (within Core Rules or a dedicated section) |
| >30 lines per topic | `references/*.md` — one file per topic, SKILL.md only has a load guide |

### Reference Files (`references/`)

Process descriptions that are too long to inline but too structured for RAG go into `references/`. This is the pattern for flow descriptions, architecture docs, step-by-step guides, and configuration checklists.

**SKILL.md writes a load guide, not the content:**

```markdown
## Reference Files
| File | Content | When to Load |
|------|---------|-------------|
| `references/order-flow.md` | Order processing pipeline (validation → payment → fulfillment) | Writing order-related code |
| `references/auth-architecture.md` | Authentication/authorization chain (login → session → RBAC) | Modifying login or permission logic |
```

**Rules for reference files:**
- One topic per file (don't create a mega-reference)
- File should be self-contained — readable without other references
- SKILL.md tells AI **when** to load each file, not just **what** is in it
- Never load all references at once — that defeats the purpose

## SKILL.md Structure Budget

Target: **300-500 lines**. If exceeding, move data to CSV.

Required sections:

```markdown
# Skill Title

## Trigger Conditions        ← when to use / skip

## Core Rules (<=20)         ← things AI must always hold in context

## Terminology Table         ← <=50 rows max
| Business Concept | Code Entity | Service |

## Reference Files           ← load guide for process descriptions (if any)
| File | Content | When to Load |

## Search Guide              ← which domain to search, what commands to use
- Available domains and their search commands
- Default result count: top 3

## Decision Rules            ← scenario → condition → strategy
| Scenario | Condition | Strategy |

## Pre-Delivery Checklist    ← verify before handoff
```

## CSV Knowledge Base Design

### Schema: Separate Search from Output

```python
{
    "domain_name": {
        "file": "data.csv",
        "search_cols": ["Name", "Keywords", "Category"],     # BM25 matching
        "output_cols": ["Name", "Detail", "Code", "Checklist"] # Returned to AI
    }
}
```

- `search_cols`: keyword-dense fields for matching
- `output_cols`: complete info needed to generate output

### Row = Self-Contained Entity

Each row must be usable independently — never cross-reference other rows:

```csv
Name,Keywords,Category,Config_Template,Validation_Rules,Common_Pitfalls
StripeWebhook,"payment,webhook,HMAC","payment-gateway","{full template}","{rules}","Watch for idempotency keys"
```

### Reasoning CSV

Create a separate reasoning CSV when Decision Rules in SKILL.md exceed 10 rows and are still growing:

```csv
Scenario,Condition,Recommended_Approach,Priority,Anti_Patterns,Decision_Rules
user_signup,has_email_verified,skip_verification_step,HIGH,no_double_opt_in,"{""if_no_email"":""require_phone_verification""}"
```

## Retrieval Layer

### Engine Selection

| CSV Rows | Engine | Notes |
|---|---|---|
| <= 5000, fixed schema | BM25 (pure Python) | Zero dependency, precise |
| Keyword mismatch expected | Vector search | Semantic fallback |
| Cross-entity relationships | Knowledge graph | Call chains, dependencies |

### Domain Routing

When multiple CSVs exist, auto-detect domain by query keywords:

```python
domain_keywords = {
    "api_config":   ["endpoint", "webhook", "auth", "token", "header"],
    "error_codes":  ["error", "status", "exception", "code", "retry"],
    "data_models":  ["schema", "field", "table", "column", "entity"],
}
# Fallback to most general domain if no match
```

### Result Control

- Default: **top 3**
- Only return results with score > 0
- Truncate fields > 300 chars
- Output format: structured markdown

## Anti-Patterns → Fix

| Do Not | Do Instead |
|---|---|
| Merge multi-domain knowledge into 1 skill | Check domain scope first — if domains have independent triggers, use retrieval layer + domain skills |
| Dump all knowledge into SKILL.md | Classify by form + scale, sink accordingly |
| Flatten process descriptions into CSV rows | Put in `references/*.md` — narratives lose meaning in CSV |
| CSV without search_cols/output_cols separation | Design schema with separate search and output fields |
| One mega-CSV for all domains | Split by domain, add keyword-based auto-routing |
| Hardcode >10 decision rules in SKILL.md | Extract to reasoning CSV |
| Use RAG for enumerable, fixed-schema knowledge | Use CSV + BM25 |
| Inline >50 terminology entries | Move to CSV with search script |
| Load all reference files at once | SKILL.md tells AI when to load each file — on demand only |
| Skip source verification | Verify all knowledge from primary source, add last_verified timestamp |

## Checklist: Before Delivering a New Skill

- [ ] **Domain scope assessed** — single domain → 1 skill; multi-domain → retrieval layer + domain skills
- [ ] Knowledge classified by **form** (entity vs process) AND **scale** (count/lines)
- [ ] Process descriptions >30 lines → `references/*.md`, not CSV rows
- [ ] SKILL.md has load guide for each reference file (when to load, not just what)
- [ ] SKILL.md <= 500 lines (rules + search guide + load guide only)
- [ ] Inline entries <= 50
- [ ] If CSV exists: separate search_cols and output_cols
- [ ] Each CSV row is self-contained
- [ ] Search script returns top 3 by default
- [ ] Domain auto-routing if multiple CSVs
- [ ] Reasoning CSV exists if decision rules > 10
- [ ] All knowledge verified from primary source
