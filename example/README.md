# Worked Example: Old-News Retention Cleanup

A small but realistic task driven through all 9 stages, with every handoff document filled in — showing what the documents look like and how the `needs_input` loop runs.

> ⚠️ This directory is **illustrative**. The schema and fields are real, but table names, row counts, timestamps, and infrastructure details are **fictional filler** (a generic Kubernetes + PostgreSQL stack). In a real run these values are produced by workers querying the actual environment.

## The task

> "The news database keeps growing — add a daily cleanup CronJob that deletes old articles."

A typical "small task with hidden ambiguity": the retention period was never stated (→ an open question at the requirements stage), and it involves DB deletion — a destructive operation (→ a `human_gate` task at the planning stage).

## File order

| File | Stage | What to look at |
|------|-------|-----------------|
| `00-intake.yaml` | triage | Routes on coarse signals without pinning requirements; destructive + low-reversibility → `full`, no fast-path |
| `01-requirements.yaml` | requirements | The `needs_input` → user answers → `approved` loop via open_questions |
| `02-environment.yaml` | environment | Read-only recon; records "no index" as an assumption instead of adding one on its own |
| `03-plan.yaml` | planning | Destructive task marked `human_gate`; dry-run first; **low reversibility → `rollback_plan` + t5 checkpoint task inserted; whole plan signed off by the user** |
| `04-execution.yaml` | execution | Actual results plus one recorded deviation; **t5 lands the restore point (`restore_point_ref`) before destructive t6 runs** |
| `045-review.yaml` | review | **Code only**: white-box look at the deletion script; f1 pins the execution stage's "lock waits" deviation to its root cause (missing index) — the layer black-box verification can't see |
| `05-verification.yaml` | verification | Independent comparison against acceptance_criteria, pass/fail per item |
| `055-observation.yaml` | observation | Time gate: a 7-day window is scheduled; verdict only after expiry; metrics held → promote |
| `06-retrospective.yaml` | retrospective | Aggregation + improvements fed forward into the backlog + user sign-off |

## Four mechanisms this example demonstrates

1. **Ambiguity bounces up**: the retention period is blocked on an open question at the requirements stage; the user answers before anything proceeds. No worker guessed, anywhere.
2. **Destructive operations converge on the orchestrator**: the deletion cutover is marked `human_gate` in the plan; the execution worker stops at it and waits for foreground confirmation instead of deleting on its own.
3. **Review ≠ verification (white-box vs black-box)**: verification passes ac1–ac3 and waves the lock-wait deviation through as "doesn't affect acceptance"; the 4.5 white-box review identifies the root cause — the delete predicate scans an unindexed column (recorded as an assumption back in the environment stage) — and feeds it into the backlog. "It runs" passed; "it's written right and will stay stable" needed review.
4. **A way back exists before anything is destroyed** (consuming the reversibility signal): `00` judges `reversibility=low`, and that signal does more than pick the route — planning therefore produces a `rollback_plan` and inserts the t5 checkpoint (pre-delete dump, including the CASCADE child table); destructive t6 `depends_on` it and is blocked until `restore_point_ref` lands; the whole plan additionally gets user sign-off. If observation later demanded a rollback, it would follow this existing plan — not an escape route invented mid-incident.
