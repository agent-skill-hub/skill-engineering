# Skill Engineering

**Your AI skill has 200 domain knowledge entries. You dumped them all into SKILL.md. Now AI performance tanks every time it loads.**

This is the #1 mistake in skill design: treating SKILL.md as a knowledge dump instead of a retrieval guide.

## The Insight

> AI doesn't need to *know* knowledge. It needs to *find* knowledge.

A well-designed skill keeps context consumption **constant** regardless of knowledge scale — 50 entries or 5,000, the SKILL.md stays under 500 lines. Rules are fixed. Data is pulled on demand.

## Architecture

```
skill/
├── SKILL.md           ← Rules + search guide + decision table (the brain)
├── references/        ← Process docs, loaded on demand (the narrative)
├── scripts/           ← BM25 search, domain routing (the retrieval)
└── data/              ← CSV/JSON, one row = one entity (the knowledge)
```

Most skills only need `SKILL.md`. The rest kicks in when scale demands it.

## The Form × Scale Framework

Not all knowledge is the same shape. A "config template" and an "auth flow diagram" are fundamentally different — yet most skill designers treat them identically and wonder why things break.

**Two questions. That's it.**

**Q1: Is it a countable entity or a process description?**

| | Countable Entity | Process Description |
|---|---|---|
| Looks like | A term, a config, an error code | A multi-step flow, an architecture chain |
| Test | Can you put it in one CSV row? | Does it lose meaning when flattened? |
| Storage | Inline → Table → CSV | Inline → `references/*.md` |

**Q2: How many?**

For entities:

| Count | Where it goes |
|---|---|
| ≤ 20 | Inline in SKILL.md as rules |
| 20–50 | Terminology table in SKILL.md |
| 50+ | `data/*.csv` + BM25 search script |

For process descriptions:

| Length | Where it goes |
|---|---|
| ≤ 30 lines | Inline in SKILL.md |
| 30+ lines | `references/*.md`, one topic per file |

## Why `references/`?

Because CSV kills narratives.

An order processing pipeline (validation → payment → fulfillment → notification) has sequential logic, branching conditions, and error recovery paths. Flatten that into a CSV row and you get garbage.

`references/` keeps narratives intact. SKILL.md holds a **load guide** — not the content itself, but *when* to load each file:

```markdown
| File | When to Load |
|------|-------------|
| `references/order-flow.md` | Writing order-related code |
| `references/auth-chain.md` | Modifying login or permissions |
```

AI loads what it needs, when it needs it. Zero waste.

## The Iron Rule

**≤ 50 entries? Keep everything in SKILL.md. Don't touch CSV. Don't create scripts.**

Most skills are Light level. The urge to over-engineer is the second most common mistake after knowledge dumping.

## What This Produces

| Before | After |
|---|---|
| 800-line SKILL.md, loaded every time | 300-line SKILL.md + on-demand data |
| Process flows crammed into CSV rows | `references/*.md` preserving full narrative |
| AI searches everything, returns noise | Domain-routed BM25, top 3 results only |
| "Add knowledge" = edit SKILL.md | "Add knowledge" = append a CSV row |
| Scale kills performance | Scale is invisible to context window |

## Quick Start

1. **Classify** your knowledge — entity or process? How many?
2. **Place** it — inline, table, CSV, or references — per the framework above
3. **Write** SKILL.md as a retrieval guide, not a knowledge dump
4. **Verify** against the [checklist in SKILL.md](./SKILL.md#checklist-before-delivering-a-new-skill)

## Who This Is For

You're building a skill that involves domain expertise — style libraries, error code catalogs, API configs, business flows, compliance rules. You need that knowledge accessible to AI without blowing up the context window.

This meta-skill gives your AI the architecture to handle it.

---

*Built for AI agents. Designed by humans who got tired of watching skills break at scale.*
