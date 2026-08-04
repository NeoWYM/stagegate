# Worker Mapping

How the autonomous stages map onto agents. Two principles drive this file:

1. **Reuse what you have.** Most stages map cleanly onto Claude Code's built-in agent types or onto custom subagents you've already built. The only stage with no generic worker is *execution* — deliberately (see "Gaps").
2. **Boundaries are tool permissions, not prompt text.** Where a stage requires read-only behavior (review, recon), enforce it by giving the agent a read-only tool list. Agents that inherit full tool access have been observed to commit, push, and even misreport their actions despite explicit "do not modify anything" instructions. An agent without `Bash`/`Edit`/`Write` *cannot* do those things.

## Worker archetypes

| Archetype | Used by stage | Tool boundary | Notes |
|-----------|---------------|---------------|-------|
| **Recon** (read-only surveyor) | 2 environment, 5 verification, 5.5 observation | Read/Grep/Glob (+ Bash only if your recon requires live queries) | Domain-specific read-only subagents (infra health, workload state, DB inspection) are ideal; built-in `Explore` is the generic fallback |
| **Planner** | 3 planning | read-only | Built-in `Plan` agent, or plan in the foreground with your planning skill |
| **Executor** | 4 execution | whatever the domain needs | **Deliberately not generic** — see Gaps |
| **Reviewer** | 4.5 review | **strictly Read/Grep/Glob** — no Bash, no Edit, no Write | Hard read-only boundary at the tool level; generate the diff yourself and paste it into the brief, since the reviewer can't run git |
| **Doc writer** | 6 retrospective (transcription only) | Read/Grep/Glob/Edit/Write | Optional; transcribes improvements/facts into the backlog. The retrospective conversation itself stays in the foreground |

## Stage → worker table

| Stage | Worker | Model tier | Notes |
|-------|--------|-----------|-------|
| 2 environment | your read-only recon subagent(s), or `Explore` | fast | Recon is read-only surveying — exactly what such agents are for. The orchestrator may **combine several** (infra + DB + IaC diff), each producing its slice; **you merge them into a single `02-environment.yaml`** and judge the gate — don't let partial documents flow downstream. **Capability probes against external services (third-party APIs, cloud permissions, quotas) are usually outside a domain recon agent's scope** — they cover your own systems; issue the minimal call yourself, or delegate `general-purpose` (fast/medium) under explicit constraints: read-only, judge by response code, never print credentials in the clear |
| 3 planning | built-in `Plan` agent (isolated context) or a foreground planning skill | strong | Require `human_gate` marks on destructive tasks; **low-reversibility/destructive → the plan must include `rollback_plan` + a checkpoint task, and the whole plan returns to the orchestrator for user sign-off** |
| 4 execution | domain agents / `general-purpose` with the task brief | fast (ordinary) / **strong** (destructive, low-reversibility, `human_gate`) | No universal executor. Domain agent if you have one; otherwise the orchestrator executes directly or delegates `general-purpose` with the stage prompt embedded. **Destructive steps blocked until the checkpoint task lands `restore_point_ref`** |
| 4.5 review | read-only reviewer agent | **strong — never downgrade** | Code artifacts only; white-box review of the code against requirement intent + quality, catching what black-box verification misses (logic, edge cases, security, tech debt). Independent of the executor |
| 5 verification | recon subagent(s) again | strong | Naturally independent of the executor. If your ops rules already say "verify with X after deploying", this stage just institutionalizes that rule |
| 5.5 observation | the same recon agent, re-run | fast | Same check as verification, differing only in **time**: the orchestrator schedules the wake-up; on window expiry the same worker re-checks |
| 6 retrospective | optional doc-writer for backlog transcription | fast | The retrospective itself is interactive (orchestrator, foreground model); only the mechanical "write these facts into the backlog / sync project docs" part is delegable |

> **Model/effort rule**: state model *and* effort explicitly on every dispatch (omitting them silently inherits the parent session's model). Judgment-dense stages (3 planning, 4.5 review, 5 verification) get the strong model at high effort — planning escalates to maximum effort for destructive/low-reversibility work. Stage 4 executes an approved frozen plan (checklist work): fast model at high effort for ordinary tasks — stages 4.5/5 provide independent strong-model gatekeeping downstream — but strong model for destructive/low-reversibility/`human_gate` tasks. Mechanical bounded stages (2 recon, 5.5 re-check, 6 transcription) run the fast model at medium effort. Agents that pin model/effort in their own definition need no override. On worker failure, escalate only with the full failure trace attached; at most 2 rounds on the same approach.

## Gaps and recommendations

1. **No generic execution worker — a feature, not a defect.** Execution is inherently domain-specific (editing k8s manifests vs writing to a DB vs an IaC apply sequence share nothing); a universal executor becomes one giant prompt that does everything badly. Instead:
   - Domain agent exists → use it.
   - Otherwise → the orchestrator executes directly, or delegates `general-purpose` with the task's stage prompt in the brief.
2. **High-ceremony destructive operations shouldn't be delegated at all.** Operations your own rules already ritualize (e.g. IaC: plan to a file → human reviews → apply that exact file) belong in the orchestrator's hands as `human_gate` tasks, not in a worker's.
3. **Composite recon needs orchestrator convergence.** One task may need multiple recon agents; each returns its own slice. You merge and judge — two half-documents must never flow downstream separately.

## In one sentence

The only genuinely new artifacts are the orchestrator skill and the per-stage briefs; the workers are almost entirely things you already have — recon for environment/verification/observation, `Plan` for planning, a tool-bounded read-only reviewer for review, and a doc-writer for retrospective transcription. The deliberate blank is the universal executor, because execution should stay domain-specific.
