# stagegate

A staged, gated task pipeline for [Claude Code](https://claude.com/claude-code) — drive risky, destructive, or long-running tasks through explicit stages (premise check → triage → requirements → environment → planning → execution → review → verification → observation → retrospective), with frozen handoff documents between stages, human gates where they matter, and a cross-iteration backlog that turns a linear run into a loop.

Ships as a Claude Code **skill**: the orchestrator instructions live in [`SKILL.md`](SKILL.md), with reference documents loaded on demand (progressive disclosure).

## Why

Agents are good at doing work and bad at knowing when to stop and ask. Left alone on a risky task, an agent will happily assume missing requirements, skip the backup, and declare success from a green test run. stagegate addresses this structurally rather than with prompt-level pleading:

1. **Interaction converges on one layer.** Only the orchestrator (the foreground session) talks to the user. Autonomous workers that hit ambiguity don't guess — they return `status: needs_input` with explicit `open_questions`, bouncing control back to the human layer.
2. **Stages hand off through frozen documents, not conversation memory.** Each stage consumes only `approved` upstream documents and produces one of its own. Context exhaustion can't lose the state; a fresh session can resume from the documents.
3. **Gates are typed.** Human gates (requirements, retrospective, and planning when the task is destructive), a route gate (triage), a time gate (observation — "passing now" is not "still passing next week"), and auto gates with mechanical exit criteria.
4. **Destructive work requires a way back before it runs.** Low-reversibility tasks force a `rollback_plan` plus a checkpoint task in planning; the destructive step is blocked until a restore point is verified restorable — not merely "the backup file exists."
5. **Capabilities are proven, not presumed.** Before planning starts, every external API, permission, and quota the requirements depend on is exercised with one minimal real call. Documentation saying a thing exists is not evidence that your account can do it — and finding out after execution means rolling back the whole lap.
6. **A backlog layer makes it a loop.** Improvements and promoted facts from each retrospective feed forward into the next iteration's triage and environment stages.
7. **Premises are checked before you spend on them.** When intent points at an existing target, one real read-only command must confirm it's still there — not retired, not superseded — before triage even starts. This runs earlier than the capability probe, because a spec-driven-development proposal step (if wired in) typically fires right after requirements sign-off, ahead of the probe; without this pre-check, that step can burn a full proposal/design/tasks set on a target that no longer exists.

## The pipeline

```text
-1   premise check  orchestrator, conditional        ── Pre-check: only when intent targets something existing
0    triage         orchestrator, interactive        ── Route gate: fast | full (defaults full)
1    requirements   orchestrator, interactive        ── Human gate: user sign-off
2    environment    worker, read-only recon          ── capability probes must pass before planning
3    planning       worker                           ── Human gate when destructive / low-reversibility
4    execution      worker                           ── destructive steps blocked until restore point lands
4.5  review         worker, code laps only           ── skipped for non-code artifacts
5    verification   worker, independent of executor
5.5  observation    worker, scheduled                ── Time gate: no verdict before observe_until
6    retrospective  orchestrator, interactive        ── Human gate: next_iteration
```

- **Premise pre-check**: runs before route is even decided, and only when intent names an existing target. `ABORT` ends the lap on the spot — no downstream document, no SDD proposal step — since a proposal step (where wired in) fires earlier than the stage-2 capability probe and can't be relied on to catch a retired or superseded target.
- **Fast-path**: triage can short-circuit small tasks past some stages — but routes only ever escalate, never downgrade, and destructive/low-reversibility tasks — and tasks depending on an external capability that has never been exercised — can never take the fast path.
- **Tight loop** (within an iteration): `needs_input`, review rework, and verification/observation failures bounce back to the appropriate upstream stage.
- **Wide loop** (across iterations): retrospective ends with `next_iteration: continue | done | abort | spawn`; improvements are already in the backlog for the next lap.

## Repository layout

| File | Contents | Read when |
|------|----------|-----------|
| [`SKILL.md`](SKILL.md) | Orchestrator operating manual (the skill entry point) | Always loaded first |
| [`stages-and-gates.md`](stages-and-gates.md) | The 9-stage state machine (plus the conditional stage -1 premise check): owners, gates, flow diagram, loop rules | You want to know **how the flow runs** |
| [`handoff-schema.md`](handoff-schema.md) | YAML schema and invariants for stage handoff documents | You want to know **how the documents are written** |
| [`backlog.md`](backlog.md) | Cross-iteration persistence layer schema | You want to know **how iterations connect** |
| [`worker-mapping.md`](worker-mapping.md) | Mapping stages to worker agents, model selection, delegation protocol | You want to know **who does the work** |
| [`example/`](example/) | A complete worked example: all 9 handoff documents for one small task | You want to see **what it actually looks like** |

Suggested reading order: the role and stage-overview sections of `stages-and-gates.md` → `example/README.md` → back to `handoff-schema.md` and `backlog.md` for field-level detail.

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/NeoWYM/stagegate.git ~/.claude/skills/stagegate
```

Or for a single project, into `<repo>/.claude/skills/stagegate`.

Then in Claude Code, trigger it naturally ("run this through the pipeline", "use the staged process for this") or start a conversation about a risky change — the skill description covers the trigger conditions.

## Adapting it to your environment

The skill is deliberately environment-agnostic. Three places you will want to customize:

- **Workers** (`worker-mapping.md`): map the recon/verification stages to your own read-only subagents if you have them (infra health checks, DB inspectors, …). Built-in `Explore` / `Plan` / `general-purpose` agents work out of the box.
- **Review boundary**: run the review stage on an agent whose *tool permissions* are read-only. Prompt-level "please don't modify anything" is not a boundary; tool allowlists are.
- **Pipeline directory**: pick a stable location for the per-iteration `NN-<stage>.yaml` documents and the `backlog.md` file (e.g. `pipelines/<task-slug>/` in your work directory).

## Design notes

- Interactive stages (0, 1, 6) are never delegated: in Claude Code, a subagent cannot hold a back-and-forth conversation with the user — it returns one final message. Anything conversational must stay in the foreground session.
- There is intentionally **no universal executor worker**. Execution is domain-specific; a one-size-fits-all executor becomes a giant prompt that does everything badly. The orchestrator either executes directly or delegates to a domain agent per task.
- Verification and review are separate stages because they answer different questions: verification is black-box ("does it work?"), review is white-box ("is it written correctly, and is it what was actually asked for?"). Green tests can coexist with hardcoded values, missed edge cases, and security problems.

## License

[MIT](LICENSE)
