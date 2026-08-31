# Handoff Document Schema

The only information interface between stages. An upstream stage produces a document conforming to this schema, freezes it, and passes it downstream; downstream reads only this document — never the upstream stage's conversation context.

Design goals:

- **Self-contained**: downstream can execute from the document alone, without seeing upstream reasoning.
- **Traceable assumptions**: `assumptions` lets downstream see what upstream assumed, and trace errors back.
- **Ambiguity bounces up**: any stuck stage uses `open_questions` to return control to the orchestrator instead of inventing an answer.

## Format

Extension `.yaml` (or markdown frontmatter). One document per stage, named `NN-<stage>.yaml` (e.g. `01-requirements.yaml`).

```yaml
# ── meta ─────────────────────────────────────────────
meta:
  doc_id: req-2026-06-16-001       # globally unique; used for cross-stage references
  stage: requirements               # intake | requirements | environment | planning
                                    # | execution | review | verification | observation
                                    # | retrospective
  schema_version: 1
  author: orchestrator              # orchestrator | worker:<agent-name> | human
  created_at: 2026-06-16T12:40:00Z
  updated_at: 2026-06-16T12:55:00Z

# ── status ───────────────────────────────────────────
# draft       : still being produced; not consumable downstream
# needs_input : stuck on open_questions; waiting for the human layer
# approved    : passed this stage's exit gate; frozen; may flow downstream
# rejected    : gate failed; bounce upstream or redo
status: approved

# ── chaining ─────────────────────────────────────────
upstream_ref:                       # which upstream documents this one consumed
  - req-2026-06-16-001
frozen_at: 2026-06-16T12:55:00Z     # filled on approval; immutable afterwards

# ── body ─────────────────────────────────────────────
summary: >
  One to a few sentences letting downstream (and humans) grasp this
  document's conclusion quickly.

content:                            # stage-specific payload; structure per stage
  # intake:
  #   raw_request / route / signals(risk, scope, known_pattern, reversibility)
  #   / skipped_stages[] / rationale
  #   + premise_check (filled only when intent points at an existing concrete target;
  #     omit or set applicable:false for a brand-new build):
  #     { applicable: bool, target: str, checks: [{claim, evidence}], verdict: PROCEED|ABORT }
  #     verdict=ABORT ⇒ this 00-intake.yaml itself is never approved; the lap ends here —
  #     no downstream document, no SDD proposal step
  #     (the failure this prevents: two rounds each burned a full recon + proposal/
  #      design/tasks set before discovering the target had already been retired or
  #      superseded — see the "-1. Premise liveness check" stage in stages-and-gates.md)
  # requirements:
  #   goal / scope / non_goals / constraints / acceptance_criteria
  # environment:
  #   current_state / gaps[] / blockers[]
  #   + capability_probes[] (name, probe_cmd, result, evidence) — for every external
  #     API/permission/quota the requirements depend on, evidence from one minimal
  #     real call; never presumed from documentation or memory. Any result other
  #     than pass ⇒ goes into blockers[] and status: needs_input
  #     (the failure this prevents: a whole lap finished before discovering the
  #      assumed API does not exist for that account tier — total rollback)
  # planning:
  #   tasks[] (id, desc, depends_on, owner, est, acceptance, human_gate)
  #   destructive/low-reversibility additionally: rollback_plan
  #     (trigger, method, restore_point, rpo, owner)
  #   + tasks[] must contain one checkpoint task (produces the restore point),
  #     with destructive tasks depends_on it
  #   + checkpoint and its later verify task must be SYMMETRIC: whatever
  #     snapshots the checkpoint captured (disk usage, object lists,
  #     restart counts, ...), the verify task must capture the same set
  #     post-change — spelled out concretely in both tasks' acceptance,
  #     not as a vague "keep snapshots". Otherwise one side gets skipped
  #     and the miss only surfaces at the exit gate.
  # execution:
  #   results[] (task_id, status, output_ref, deviation)
  #   destructive tasks additionally: restore_point_ref
  #     (backup file / snapshot ID / plan-file path)
  # review (code artifacts only):
  #   findings[] (severity, location, issue, against_requirement)
  #   verdict (pass | rework)
  # verification:
  #   checks[] (id, expected, actual, pass)
  # observation:
  #   observe_until / sample_threshold / metrics[] (name, baseline,
  #   observed, hold) / decision (promote | rollback)
  ...

# ── assumptions ──────────────────────────────────────
# Every assumption this stage made in order to proceed.
# Visible to downstream stages and to humans.
assumptions:
  - id: a1
    text: "Target is the existing cluster; no new nodes needed"
    confidence: high               # high | medium | low
    source: "confirmed by user during requirements"   # user / inferred / existing docs

# ── open questions (the ambiguity bounce-up mechanism) ──
# Anything this stage is stuck on, unsure about, or must not guess.
open_questions:
  - id: q1
    question: "Retention: 7 days or 30 days?"
    options: ["7 days", "30 days", "tiered by data class"]  # empty if free-form
    blocking: true                 # true = blocks stage progress; false = note for later
    answer: "30 days"              # filled once answered
    answered_by: human             # human | orchestrator
    answered_at: 2026-06-16T12:52:00Z

# ── decisions ────────────────────────────────────────
# Gate passes, sign-offs, and resolved open questions — the audit trail.
decisions:
  - at: 2026-06-16T12:55:00Z
    by: human
    decision: "Requirements signed off; frozen; entering environment stage"

# ── exit checklist ───────────────────────────────────
# This stage's exit-gate checklist; all true before approved.
acceptance:
  - { item: "scope and non_goals explicit", done: true }
  - { item: "no blocking open_questions", done: true }
  - { item: "acceptance criteria measurable", done: true }
```

## Invariants

When reading or writing these documents, the following must always hold:

1. Any `blocking: true` open question with an empty `answer` ⇒ `status` **must be** `needs_input`.
2. `status: approved` ⇒ all `acceptance.done` true, and `frozen_at` filled.
3. After `approved` the document is **frozen** — no further edits; changes require a new version (`doc_id` incremented or `-v2` suffixed) that re-passes the gate.
4. Downstream may consume only upstream documents with `status: approved`.
5. Every `assumption` must have a `source`; a high-risk assumption with `source: inferred` should be considered for promotion to an `open_question`.

## Why this design

- `open_questions` + invariant 1 turn "a human is needed here" into **an explicit document-level state**, rather than something buried inside an agent. Every worker uses the same mechanism to bounce questions to the human layer → intervention points converge and become predictable.
- `assumptions.source` distinguishes "user confirmed" from "I inferred" — directly encoding the rule against inventing requirements: if an inferred assumption matters, it should have been an open question.
- Freezing + versioning prevents upstream documents from being quietly edited while downstream is mid-flight, keeping the pipeline reproducible.
