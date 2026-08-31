# Stage Definitions and Gate Flow

The pipeline is a state machine. Every stage has an explicit **owner** (who executes it), **input/output documents**, an **entry condition**, and an **exit gate**. Document format is defined in `handoff-schema.md`.

## Roles: Orchestrator vs Worker

- **Orchestrator (human-interface layer)**: the foreground session — the only role that can talk to the user. Runs the interactive stages, collects answers to `open_questions`, enforces gates, chains the stage documents.
- **Worker (autonomous stages)**: an independent agent/session with a clean context and a stage-specific prompt. Consumes one approved document, produces one new document. **Never touches the user** — when stuck it outputs `status: needs_input`, bouncing control back to the orchestrator.

> Core constraint: "talking to the human" always converges on the layer that has user access. In Claude Code a subagent cannot hold a back-and-forth with the user — it returns a single final message. So interactive stages are always run by the orchestrator itself, never delegated.

## Two kinds of state: per-iteration vs backlog

- **Per-iteration documents** (`NN-<stage>.yaml`): handoffs within a single lap; frozen and archived when the lap ends. The state machine's transient state.
- **Backlog layer** (`backlog.md`): **persists across iterations**, independent of any lap. Holds open improvements, assumptions promoted to standing knowledge, recurring environment facts, and lap exit decisions. Retrospective feeds improvements forward into it; triage and environment read it back on the next lap. Format in `backlog.md`.

> Without the backlog layer the pipeline is a "linear one-shot". With it, retrospective output reconnects to the next lap's input, and the linear skeleton becomes an actual loop.

## Stage overview

| # | Stage | Owner | Input | Output | Gate type |
|---|-------|-------|-------|--------|-----------|
| -1 | premise liveness check | **Orchestrator** | user intent, `backlog`/memory | folded into `00-intake.yaml` | Pre-check (conditional) |
| 0 | triage | **Orchestrator** | user intent, `backlog` | `00-intake.yaml` | Route gate |
| 1 | requirements | **Orchestrator** | `00` | `01-requirements.yaml` | **Human gate** |
| 2 | environment | Worker | `01`, `backlog` | `02-environment.yaml` | Auto |
| 3 | planning | Worker | `00`, `01`, `02` | `03-plan.yaml` | Auto / **Human** (destructive/low-reversibility) |
| 4 | execution | Worker | `03` | `04-execution.yaml` | Auto |
| 4.5 | review (code only) | Worker (strong model) | `01`, `03`, `04` | `045-review.yaml` | Auto (conditional) |
| 5 | verification | Worker | `01`, `04` | `05-verification.yaml` | Auto |
| 5.5 | observation | Worker (scheduled) | `01`, `05` | `055-observation.yaml` | Time gate |
| 6 | retrospective | **Orchestrator** | everything, `backlog` | `06-retrospective.yaml` | **Human gate** |

Interactive stages (0, 1, 6) are run by the orchestrator; 2–5.5 are autonomous workers. **-1 is a conditional pre-check**, triggered only when intent points at an existing concrete target; the orchestrator does it directly, no separate worker.

**Triage can short-circuit**: the `route` in `00` decides between the full 9 stages and a fast-path (e.g. a single non-code parameter change may skip 2/3/4.5/5.5 and run only 1→4→5). Not every lap deserves full weight.

## Overall flow

```text
             (triage reads improvements + standing_facts from the backlog)

user <──> 0. triage ─────────────── route=fast ───────────────┐
              │ route=full                                     │
              ▼                                                │  fast-path skips
user <──> 1. requirements ◄────────────────┐                   │  2 / 3 / 4.5 / 5.5
              │ approved                   │                   │  (never allowed for
              ▼                            │                   │  destructive work)
          2. environment (worker) ─────────┤                   │
              │ approved                   │                   │
              ▼                            │  needs_input:     │
          3. planning (worker) ────────────┤  bounce up to     │
              │ approved                   │  orchestrator,    │
              ▼                            │  ask the user,    │
          4. execution (worker) ◄──────────┤  merge answers,   │
              │ done                       │  re-run stage.    │
              ▼                            │                   │
          4.5 review (worker, code only) ──┤  re-route:        │
              │ pass          │            │  escalate and     │
              │               └─ rework ─► │  back-fill        │
              ▼                  back to 4 │  skipped stages.  │
          5. verification (worker) ◄───────┴───────────────────┘
              │ pass          └─ fail ─► back to 4 (or 1)
              ▼
          5.5 observation (worker, scheduled)  ⏰ wakes after observe_until
              │ held          └─ fail ─► back to 4 (or 1), rollback per 03's plan
              ▼
user <──> 6. retrospective ── writes improvements / facts / log to the backlog
              │
              ├── continue ──► new lap (back to 0, backlog feeds forward)
              ├── done / abort ──► end
              └── spawn ──► fork an independent task
```

- **Tight loop (within a lap)**: `needs_input` bounce-ups and verify/observation failures return to the previous working stage. Fixed within the same lap.
- **Wide loop (across laps)**: retrospective sets `next_iteration=continue` → back to triage for a new lap, with improvements already in the backlog.
- Whenever any worker raises `needs_input`, control returns to the orchestrator → ask the user → merge answers → re-run that stage. All human intervention points converge on the orchestrator.

## Stage details

### -1. Premise liveness check 〔Pre-check, conditional〕

- **Owner**: Orchestrator (before any interaction, very lightweight — usually one real command, no worker session).
- **Trigger condition**: the user's intent **points at an existing, concrete target** (a feature, pipeline, CronJob, table, or known service) — not a brand-new build. A new feature has no "existing premise" to verify, so this check does not fire.
- **Entry**: user has stated intent; not yet in `0. Triage`.
- **What it does**: with **one minimal, real, read-only command** (not a doc lookup, not memory), confirm:
  1. The target code path / CronJob / table / service **still exists and has not been retired or superseded** (e.g. `kubectl get cronjob`, grep the routing in code, check backlog/memory for a "retired" note).
  2. Backlog/memory has no existing record that contradicts this intent (e.g. the same thing was already closed, or already replaced by another feature).
  - If it's unclear whether the target is still alive, **do not** skip the check on the assumption "it's probably still there" — that assumption is exactly what caused two prior rounds to burn a full set of proposal/design/tasks artifacts before discovering the premise was invalid.
- **Exit gate**: emit `PROCEED` or `ABORT`, with evidence (command output / file path).
  - `PROCEED` → continue normally into `0. Triage`; the evidence is recorded in `00-intake.yaml`'s `content.premise_check` (see `handoff-schema.md`).
  - `ABORT` → report the finding to the user on the spot. **Do not create any artifact downstream of `00-intake.yaml`, and especially do not trigger an SDD/spec-driven-development proposal step** — the proposal/design/tasks artifacts that step produces are exactly the sunk cost this check exists to avoid. If the user confirms a different target/scope, re-run this check.
- **No separate YAML needed**: the verdict is folded into `00-intake.yaml`; without a `PROCEED` there is no such document.

> Why this sits before triage rather than inside it: an SDD proposal step (where one is wired in) typically fires right after requirements sign-off — earlier than stage 2's environment capability probe. Whether the premise is still alive and whether an external capability is available are two different questions; the capability probe cannot be relied on to catch "the target isn't there anymore." Placing this check ahead of the entire pipeline, earlier than triage itself, is what stops the sunk cost before a proposal step ever fires.

### 0. Triage 〔Route gate〕

- **Owner**: Orchestrator (interactive, but very lightweight).
- **Entry**: the user expresses an intent.
- **What it does**: routing **without pinning down requirements** — like emergency-room triage: no diagnosis, just a severity estimate from surface signals. The judgment inputs are **coarse signals**, not finalized requirements:
  - Risk / blast radius: destructive? touches production / running systems / data deletion?
  - Scale: clearly a point change (one parameter, one schedule) vs open-ended ("optimize…")?
  - Known pattern: does the backlog hold a same-shape precedent?
  - Reversibility: if it goes wrong, how hard is the way back?
- **Output**: `route` (`fast` | `full`) + rationale. **Provisional, defaults heavy** — when in doubt, `full`.
- **Exit gate (Route gate)**: route decided. `fast` → a reduced subset (e.g. 1→4→5, skipping 2/3/4.5/5.5); `full` → all 9 stages (4.5 still conditional on the artifact being code). **A task depending on an external API / permission / quota that has never been exercised may not take `fast`** — the fast path skips stage 2, so there is no capability probe, and a capability that turns out not to exist only blows up after execution. **Destructive or low-reversibility tasks may never take `fast`** — the fast path skips planning, so there would be no checkpoint task, and the recovery precondition (rule 10) would deadlock execution. This is the hard reason "destructive or low-reversibility ⇒ heavy", not mere conservatism.
- **Re-route (critical)**: the route is not final. Any downstream stage discovering "looked small, isn't" can **escalate** the route and back-fill the skipped stages. Downgrading is never allowed (you may only move toward more rigor, never less).
- **Required `content` fields**: `route` / `signals` (risk, scope, known_pattern, reversibility) / `skipped_stages[]` / `rationale`.

> Why triage doesn't need requirements first: triage decides *how much process this deserves*, not *what this is*. The former needs only coarse signals; the latter is the requirements stage's job. With provisional + escalate-only, an early misjudgment is safe — defaulting heavy means errors cost extra process, never a skipped safeguard.

### 1. Requirements 〔Human gate〕

- **Owner**: Orchestrator (interactive). **Do not make this a worker.**
- **Entry**: user intent exists.
- **What it does**: forces vague intent into executable requirements. Produce a draft with `status: needs_input` + `open_questions`, put them to the user, loop until cleared.
- **Exit gate**: `scope`/`non_goals`/`constraints`/`acceptance_criteria` complete, no blocking open questions, explicit user sign-off → `approved` + frozen.
- **Required `content` fields**: `goal` / `scope` / `non_goals` / `constraints` / `acceptance_criteria[]`.

### 2. Environment 〔Auto〕

- **Owner**: Worker (read-only recon — infra health checks, workload state, schema inspection, file reading).
- **Entry**: `01` approved.
- **What it does**: surveys the gap between current state and the requirements (versions, capacity, permissions, existing resources). Makes no changes.
  - **Capability probe (mandatory)**: list every external API / permission / quota / resource the requirements depend on, and **prove each one usable with a single minimal real call** — never presume it from official documentation, memory, or "it should be there". Probes are read-only and must not mutate state; when credentials are involved, judge success by the response code alone and never have the worker print the secret in the clear (read-only tooling does not stop an agent from reading a secret out and echoing it into a document).
- **Exit gate**: gap survey complete **and every probe passes**. Any probe that fails or cannot be run → record it in `blockers[]` and bounce up with `needs_input`; it **must not proceed to planning**. If requirements conflict with reality (e.g. an assumed resource doesn't exist) → likewise open a blocking open_question.
- **Required `content` fields**: `current_state` / `gaps[]` / `blockers[]` / `capability_probes[]` (name, probe_cmd, result, evidence).

> Why the probe is separate from the gap survey: the gap survey asks "how far is reality from the requirements", and in doing so presumes the capability exists; the probe asks "does this capability exist at all". The failure mode this prevents is a whole lap — research, implementation, self-verification, all complete — collapsing at the end because the API endpoint the requirement assumed simply does not exist for that account tier, forcing a total rollback. One real call inside the first fifteen minutes stops the entire sunk cost at the door. The same shape recurs whenever a tool or framework is evaluated on its documentation and only measured after it has been adopted.

### 3. Planning 〔Auto, may needs_input; destructive/low-reversibility → Human〕

- **Owner**: Worker (`Plan` agent or a planning skill).
- **Entry**: `00` (for `signals.reversibility`), `01`, `02` all approved.
- **What it does**: decomposes tasks, sets dependencies, orders work. If planning uncovers remaining ambiguity in the requirements → open_question bounce-up; **never decide alone**.
  - **Destructive / low-reversibility tasks** (`00.signals.reversibility=low` or any destructive task): must additionally output a **`rollback_plan`**, and insert a **checkpoint task** into `tasks[]` (produces the restore point: dump / snapshot / plan file), with **every destructive task `depends_on` that checkpoint**.
  - This is where the `reversibility` signal from `00` is consumed — low reversibility isn't just "go full route", it forces a way back to exist first.
- **Exit gate**: every task has an owner and an acceptance check; the dependency graph is acyclic; **if destructive/low-reversibility, `rollback_plan` is complete with its checkpoint task**.
  - **Conditional human gate**: full route + destructive/low-reversibility tasks → exit escalates to a human gate; the user signs off on "the overall plan and the way back" (not just per-task confirmation later). Simple tasks stay Auto.
- **Required `content` fields**: `tasks[]` (id, desc, depends_on, owner, est, acceptance, `human_gate`). Destructive/low-reversibility additionally `rollback_plan` (trigger, method, restore_point, rpo, owner).

### 4. Execution 〔Auto〕

- **Owner**: Worker (task-dependent; possibly several in parallel).
- **Entry**: `03` approved.
- **What it does**: executes the plan, recording each task's actual result and deviations. **Destructive operations still route through the orchestrator for confirmation** — such steps were marked `human_gate` at planning time.
  - **Checkpoint first**: before a destructive task runs, its checkpoint dependency must be complete with the restore point written into `restore_point_ref` (backup file / snapshot ID / plan-file path). **No restore point ⇒ the destructive step is blocked** — never "do it first".
- **Exit gate**: all non-blocked tasks done; destructive tasks have `restore_point_ref` filled; failures or plan deviations recorded, `needs_input` if warranted.
- **Required `content` fields**: `results[]` (task_id, status, output_ref, deviation; destructive tasks also `restore_point_ref`).

### 4.5 Review (code artifacts only) 〔Auto, conditional〕

- **Owner**: Worker, **independent of the executor** (no player-referee). Use a reviewer whose **tool permissions are read-only** (read/grep/glob only — no shell, no edit, no write). Prompt-level "do not modify anything" is not a boundary; agents inheriting full tool access have been observed to commit and push despite explicit instructions. Tool allowlists are the boundary. Run it on your **strongest model** — this stage's whole value is reasoning depth; downgrading it defeats its purpose. If the reviewer can't run git itself, generate the diff first and paste it into the brief.
- **Trigger condition**: this lap's artifact is code. Non-code (pure parameter/schedule/config changes) or fast-path → skipped automatically, without blocking the flow.
- **Entry**: `04` complete; `01` (requirement intent) and `03` (original plan) available.
- **What it does**: **white-box** examination of the code itself — logic correctness, edge cases, security (hardcoded secrets / unsafe APIs), whether it's genuinely what the requirement asked for vs merely passing acceptance literally, tech debt and abstraction damage. **This is the layer black-box verification cannot see.**
- **Exit gate**: every finding has a severity and a location; no blocking findings → `approved`, on to 5; blocking findings → `rework`, back to 4 (or to 1 if the finding points at ambiguity in the requirements themselves).
- **Required `content` fields**: `findings[]` (severity, location, issue, against_requirement) / `verdict` (pass | rework).

> Verification answers "does it run?"; review answers "is it written right, and is it the thing that was wanted?". All tests green ≠ correct code — hardcoding, acceptance gaming, missed edge cases, and security holes all pass black-box checks. For code tasks, skipping this stage means waving through workmanship unexamined.

### 5. Verification 〔Auto〕

- **Owner**: Worker, **independent of the executor**.
- **Entry**: `01` (for acceptance criteria) and `04` (for actual results) available.
- **What it does**: compares each `acceptance_criteria` item against actual results; outputs pass/fail per item.
- **Exit gate**: every criterion has expected/actual/pass. All pass → approved; any fail → record it, usually back to 4 (or to 1 if the requirement itself was the problem).
- **Required `content` fields**: `checks[]` (criterion_id, expected, actual, pass).

### 5.5 Observation 〔Time gate〕

- **Owner**: Worker (schedule-triggered re-check after N days); the orchestrator hangs the wake-up (short window → an in-session scheduler like `ScheduleWakeup`; long window → a cron/scheduled task).
- **Entry**: `05` approved (verification passed *now*). Fast-path may skip.
- **What it does**: **"passes now" ≠ "still holds after days of runtime"**. After the agreed observation window (N days + a sample threshold, e.g. 14 days / 20 events), re-check whether the metrics held.
- **Exit gate (Time gate)**: the verdict may only be made after `observe_until` expires **and** the sample threshold is met. Held → `approved`, promote; didn't hold → record, back to 4 (or 1), and roll back if warranted (**executing `03`'s `rollback_plan`** — never an improvised escape route).
- **Required `content` fields**: `observe_until` / `sample_threshold` / `metrics[]` (name, baseline, observed, hold) / `decision` (promote | rollback).

### 6. Retrospective 〔Human gate〕

- **Owner**: Orchestrator (interactive).
- **Entry**: `05` (or `055` if there was an observation window) complete.
- **What it does**: aggregates the lap's assumptions, deviations, failed checks, and observation results; decides with the user whether to accept / open improvements / start a new iteration. **Improvements and promoted facts feed forward into `backlog.md`.**
- **Exit gate**: explicit user sign-off on "accept or next step"; `next_iteration` set to one of `continue` | `done` | `abort` | `spawn`.
- **Required `content` fields**: `outcomes[]` / `improvements[]` (synced to backlog) / `next_iteration`.

## Universal gate rules

1. **Entry**: all listed upstream documents must be `status: approved`; otherwise blocked.
2. **Exit**: all `acceptance` items `done: true` before `approved`; otherwise stay `draft`/`needs_input`/`rejected`.
3. **Human gates (1, 6; and 3 when destructive/low-reversibility, see rule 11)**: exit requires a `decisions[]` entry recording user sign-off.
4. **Route gate (0)**: exit produces `route`, provisional and defaulting to `full`; downstream may only escalate, never downgrade.
5. **Time gate (5.5)**: verdicts only after `observe_until` expiry and sample threshold; no early approval.
6. **Conditional stage (4.5)**: runs only when this lap's artifact is code; non-code or fast-path skips it without blocking. The reviewer must be independent of the executor and run on the strong model.
7. **needs_input bounce-up**: any blocking open question ⇒ immediate `needs_input`, control returns to the orchestrator. **Continuing on an invented assumption is forbidden.**
8. **Freezing**: approved documents are immutable; changes require a new version re-passing the gate (possibly invalidating downstream documents, which then re-run).
9. **Bounce-backs (tight loop)**: review rework / verification / observation failure → back to 4 (or 1 if it points at the requirements); environment blocker → back to 1; re-route escalation → back-fill skipped stages. A bounce-back is not failure — it's resolving ambiguity at the stage that owns it. **Rollback decisions (verification/observation) execute `03`'s `rollback_plan`; never invent a new escape route on the spot.**
10. **Recovery precondition (destructive/low-reversibility)**: `00.signals.reversibility=low` or any destructive task → planning must output a `rollback_plan` including a checkpoint task; execution's destructive steps are released only after the checkpoint completes (`restore_point_ref` filled), otherwise blocked. **The reversibility signal from `00` does more than pick the route.**
    - **Restore points must be provably restorable, not merely present**: the checkpoint's acceptance must demonstrate restorability (e.g. a dump that lists cleanly, a snapshot that mounts, a plan file that applies). "The file exists" ≠ "it can save you".
    - **Destructive/low-reversibility never takes fast-path**: fast-path skips planning (3), so no checkpoint task would exist and rule 10 would deadlock execution. Hence such tasks are always `full` — the route gate must not emit `fast` for them.
11. **Plan sign-off (conditional human gate, stage 3)**: full route + destructive/low-reversibility tasks → planning's exit becomes a human gate; `decisions[]` must record the user's sign-off on "the overall plan and the way back" (distinct from per-task `human_gate` confirmations at execution time). Simple tasks stay Auto.
12. **Continuation (wide loop)**: `next_iteration=continue` → back to stage 0 for a new lap, improvements already in backlog; `done`/`abort` → end; `spawn` → fork an independent new task.
13. **Premise liveness check (-1, conditional)**: when intent points at an existing concrete target, a `PROCEED`/`ABORT` verdict must be produced before `0. Triage` begins; `ABORT` ends the lap right there — no artifact downstream of `00` may be produced, and no SDD/spec-driven-development proposal step may fire. A brand-new build never triggers this check and goes straight into `0. Triage`.

## Grounding in Claude Code

- **Orchestrator** = the main foreground session; interact via `AskUserQuestion` or plain conversation to collect open-question answers.
- **Worker** = the `Agent` tool (built-in types like `Plan`/`Explore`/`general-purpose`, or your own custom subagents) or a separate session; outputs land in the pipeline directory.
- **Review worker (4.5)** = an agent whose tool list is read-only by construction, on the strong model. Only for code laps.
- **Handoff documents** = `NN-<stage>.yaml` in the pipeline directory — the state machine's persisted state (survives context exhaustion).
- **Backlog layer** = `backlog.md`, persistent across laps; written by retrospective, read by triage/environment.
- **Observation scheduling** = `ScheduleWakeup` (short) or a cron/scheduled task (long, multi-day windows); on expiry, trigger the recon worker to re-check.
- Worker delegation is expensive → a single file read or a single query doesn't deserve a worker; delegate only for context isolation or a dedicated stage prompt. Triage's fast-path exists exactly for this.
