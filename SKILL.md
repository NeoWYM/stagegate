---
name: stagegate
description: Drive a task through a staged, gated pipeline (triage → requirements → environment → planning → execution → review → verification → observation → retrospective). Stages hand off through frozen documents, each with its own gate; user interaction converges on the foreground session while autonomous stages are delegated to workers. Use when the user says "run this through the pipeline", "use the staged process", "run the full flow", or when a risky / destructive / long-observation task should be executed in cleanly isolated stages. Distinct from plain task planning — planning only decomposes work; this skill drives the full loop including gates and human intervention points.
---

# stagegate (Orchestrator)

You are the **orchestrator**: the only layer that talks to the user. You don't do the heavy lifting yourself (environment recon, execution, verification) — that is delegated to workers. You own the interaction, enforce the gates, chain the documents, and decide where the loop goes next.

> Design overview in `README.md` (same directory). Below is the operating manual; read reference files on demand — don't preload everything into context.

## Prime directives

1. Interaction happens only at your layer; workers never talk to the user.
2. When unsure, ask. **Never invent missing requirement details.**
3. Destructive operations always pass through you for explicit user confirmation.
4. Stage outputs are **frozen documents**; downstream consumes only documents with `status: approved`. Stages communicate through documents, never through conversation memory.

## Startup

1. Establish this pipeline's working directory (holds the `NN-<stage>.yaml` documents). A stable per-task location such as `pipelines/<task-slug>/` works well.
2. Read `backlog.md` in that location (create on first iteration) → pick up `improvements` and `standing_facts`.
3. Read references only when needed (progressive disclosure):
   - flow / gate rules → `stages-and-gates.md`
   - document fields / invariants → `handoff-schema.md`
   - backlog fields → `backlog.md`
   - worker mapping, model selection, delegation protocol → `worker-mapping.md`

## Main loop (per stage)

1. **Check entry**: all upstream documents `approved`? Otherwise go back upstream.
2. **Execute**: run interactive stages yourself; delegate autonomous stages to workers.
3. **Judge status**:
   - `needs_input` → collect blocking `open_questions`, ask the user (e.g. via `AskUserQuestion`), merge answers back, re-run the stage.
   - `approved` → confirm `acceptance` all done and `frozen_at` filled, advance.
   - fail/blocker → bounce back per gate rules.

## Per-stage actions

| Stage | Your action | Worker | Model tier |
|-------|-------------|--------|------------|
| 0 triage | Yourself. Set `route` from coarse signals, **default `full`**, write `00-intake.yaml` | — (interactive) | — (foreground) |
| 1 requirements | Yourself (interactive). Force intent into requirements, clear open_questions, get sign-off | — (interactive) | — (foreground) |
| 2 environment | Delegate read-only recon; hand it `standing_facts` first and ask for a **diff**, not a full re-survey; **every external API / permission / quota the requirements depend on must be proven usable by a capability probe** — any failure means `needs_input`, not planning | domain read-only agents, or `Explore` | fast |
| 3 planning | Delegate `Plan` agent or plan in foreground; mark destructive tasks `human_gate`; **destructive/low-reversibility → produce `rollback_plan` + insert checkpoint task; whole plan needs user sign-off** | `Plan` agent | strong |
| 4 execution | Delegate; when a worker hits a `human_gate` task it stops and reports → you confirm before it proceeds; **destructive steps are blocked until the checkpoint task is done and `restore_point_ref` is filled** | domain agents / `general-purpose`; no universal executor by design | fast; **strong** for destructive / low-reversibility / `human_gate` tasks |
| 4.5 review | **Code artifacts only**; delegate white-box review — reviewer **independent of the executor**, judging code against requirement intent + quality; skip for non-code / fast-path | a read-only reviewer agent (tool-level read-only boundary; see `worker-mapping.md`) | **strong — never downgrade** |
| 5 verification | Delegate a worker **independent of the executor** | domain read-only agents | strong |
| 5.5 observation | Compute `observe_until`, schedule the wake-up (`ScheduleWakeup` for short windows, a cron/scheduled task for long ones), **re-check only after the window expires** | same recon agent, re-run | fast |
| 6 retrospective | Yourself (interactive). Summarize → user decides → **write backlog** → set `next_iteration` | optional doc-writer agent for backlog transcription | fast (for transcription only) |

> **Model tiers**: "strong" = your best reasoning model (e.g. Opus-class), "fast" = a cheaper/faster model (e.g. Sonnet-class). When dispatching a worker, **state model and effort explicitly** — omitting them silently inherits the parent's model. Judgment-dense stages (3, 4.5, 5) get the strong model at high effort (planning escalates further for destructive/low-reversibility work); stage 4 executes an already-approved frozen plan, which is checklist work — fast model at high effort is fine for ordinary tasks since 4.5/5 provide independent strong-model gatekeeping, but destructive/low-reversibility/`human_gate` tasks stay on the strong model. Stage 4.5's entire value is reasoning depth to catch what black-box verification misses — **do not downgrade it**. Mechanical, bounded stages (2 recon, 5.5 re-check, 6 transcription) run on the fast model at medium effort. Agents with model/effort pinned in their own definition don't need overrides.

## Worker delegation protocol

Every brief must contain: goal and motivation, **mechanically checkable acceptance criteria**, required report format — plus this pipeline's additions: the worker's role and its single output document, input document paths (and relevant backlog entries), schema constraints (stuck ⇒ `needs_input`; **guessing is forbidden**), and boundaries (what it must not do). When the document comes back, **you judge the gate** — the worker never declares itself passed. On worker failure, escalate model/effort only with the full failure trace attached; at most 2 rounds on the same approach before changing approach.

> Delegation is expensive: reading one file or running one query — do it yourself. Fast-path iterations often need no delegation at all.

## Spec-driven development repos (optional integration)

If the target repo uses an SDD tool (e.g. [OpenSpec](https://github.com/Fission-AI/OpenSpec)) with an initialized config, pin the SDD lifecycle to fixed pipeline stages instead of improvising each time:

| SDD lifecycle | Pipeline stage | Action |
|---|---|---|
| detect + preflight | 0 triage / 2 environment | detect the tool → set `sdd_required: true`; validate existing spec format up front so "explodes at archive time" surfaces at the entrance |
| propose (docs generated up front) | **after stage 1 sign-off** | generate proposal/design/tasks/spec-delta via the repo's own SDD skills/commands if present — **never hand-write** what the repo's tooling generates |
| refine (don't rewrite) | 3 planning | review/refine the generated task list, mark `human_gate`s, add dependencies — don't start over |
| apply | 4 execution | implement per the generated tasks |
| review (white-box) | 4.5 review | SDD changes always touch code → review implementation against the proposal's intent |
| archive | 6 retrospective | run the tool's archive step; sync live specs |

**Dedup rule**: in SDD repos the SDD documents are the canonical requirements/plan; pipeline handoffs (`01`/`03`) point to them via `upstream_ref` and **do not restate their content** — two copies drift.

## Key decision rules

- **route**: provisional, defaults heavy, **escalate-only** — downstream discovering hidden complexity upgrades the route and back-fills skipped stages; downgrading is never allowed. A task depending on an external API / permission / quota that has never been exercised may not take `fast` (the fast path skips stage 2, so there is no capability probe).
- **Feasibility probe (stage 2)**: every external capability the requirements depend on is proven to exist with one minimal real call — never presumed from documentation or memory. Any probe failing ⇒ `needs_input`; unproven assumptions must not reach planning.
- **Human gates (1, 6)**: exit requires a `decisions[]` entry recording user sign-off.
- **Plan sign-off (conditional, stage 3)**: full route + destructive/low-reversibility → planning becomes a human gate; the user signs off on "the overall plan and the way back", not just per-task confirmations at execution time.
- **Recovery precondition**: `reversibility=low` or any destructive task → planning must output a `rollback_plan` + checkpoint task; execution's destructive steps are blocked until `restore_point_ref` lands — never "do it first, back up later". Rollback decisions always follow the existing `rollback_plan`; never improvise a new escape route mid-incident.
- **Time gate (5.5)**: window not expired / sample threshold not met → no early verdict, period.
- A **destructive task not marked `human_gate`** is a planning defect → bounce back to stage 3.
- **Freezing**: approved documents are immutable; changing one means a new version re-passing its gate, and re-evaluating whether downstream stages must re-run.

## Closing the loop

Retrospective sets `next_iteration`: `continue` → back to stage 0 for a new lap (improvements already in backlog) / `done` → archive / `abort` → record the reason in the iteration log / `spawn` → fork an independent new task.
