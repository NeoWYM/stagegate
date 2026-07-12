# Backlog: the Cross-Iteration Persistence Layer

Long-lived state independent of any single lap. Per-iteration `NN-<stage>.yaml` documents freeze when their lap ends; the backlog **survives across laps** — it's the wire that connects retrospective output back to the next lap's input. Without it, the pipeline is a linear one-shot.

It stores what the next lap should know but shouldn't have to rediscover.

## What it holds (three kinds)

1. **improvements**: items fed forward by retrospectives, not yet acted on — candidate inputs for the next lap's triage.
2. **standing_facts**: verified environment facts and assumptions promoted to standing knowledge, letting the environment stage **diff against a baseline instead of re-surveying from scratch**.
3. **iteration_log**: each lap's exit decision and one-line conclusion — the loop's timeline.

## Who reads, who writes

| Stage | Backlog operation |
|-------|-------------------|
| 0 triage | **reads** improvements (candidates for the next lap), standing_facts (risk estimation / known patterns) |
| 2 environment | **reads** standing_facts → reports only *deltas from last time*, doesn't re-verify what hasn't changed |
| 6 retrospective | **writes** improvements (newly opened), standing_facts (promoted this lap), iteration_log (this lap's conclusion) |

> Only the retrospective stage writes. This ensures every change to cross-lap state passes through a human gate — no mid-pipeline worker can quietly rewrite standing knowledge.

## Format

```yaml
# ── improvement backlog (candidates for the next lap's triage) ──
improvements:
  - id: imp-2026-06-16-01
    text: "articles.published_at lacks an index; deletes will slow as data grows"
    from_iteration: news-retention      # which lap raised it
    priority: low                        # low | medium | high
    status: open                         # open | scheduled | done | dropped
    promoted_to: null                    # doc_id of the lap that picked it up

# ── standing facts (the environment stage's diff baseline) ──
standing_facts:
  - id: sf-news-schema
    text: "Main article table is news.articles; article_sources FK is ON DELETE CASCADE"
    confidence: high
    verified_at: 2026-06-16
    source_iteration: news-retention
    supersedes: null                     # which older fact this replaces (facts evolve)
  - id: sf-news-cleanup
    text: "news-cleanup CronJob daily at 04:00 local; 30-day retention; BATCH=2000"
    confidence: high
    verified_at: 2026-06-16
    source_iteration: news-retention
    supersedes: null

# ── iteration timeline ───────────────────────────────
iteration_log:
  - iteration: news-retention
    closed_at: 2026-06-23T05:10:00Z
    next_iteration: done                 # continue | done | abort | spawn
    summary: "Cleanup CronJob live and verified; index optimization opened as imp-2026-06-16-01"
```

## Invariants

1. **Only the retrospective stage writes**; every other stage is read-only.
2. `standing_facts` is a **snapshot of current reality**: when a fact changes, chain it with `supersedes` — never rewrite history in place (keeps it traceable).
3. `improvements.status` flows `open` → `scheduled` (picked up by a lap's triage, `promoted_to` filled) → `done`/`dropped`.
4. Every closed lap **must** append one `iteration_log` entry; its `next_iteration` is the wide loop's exit decision.
5. The backlog grows forever; periodically archive `done`/`dropped` improvements and superseded facts to an appendix, keeping only active entries in the main file.

## Relation to other memory mechanisms

If your agent setup already has a persistent memory system, this layer is conceptually the same thing, scoped to one pipeline's iteration semantics:

- `standing_facts` ≈ reference/project memory (environment knowledge).
- `iteration_log` ≈ per-change observation/closure records.
- `improvements` ≈ "follow-up, not yet scheduled" entries.

You can either implement the backlog *on top of* your memory mechanism, or keep it a standalone file and periodically promote stable standing facts into global memory — depending on whether you want this pipeline's knowledge to leak outward.
