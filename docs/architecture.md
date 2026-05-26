# Architecture

This document is the canonical reference for the orchestrator's design. The original design discussion is preserved at `~/.claude/plans/synthetic-churning-goose.md` and is the authoritative source if anything here drifts.

## One-paragraph summary

The orchestrator drives a single rep through five gated phases (scope → design → build → test → deploy). Between phases, a panel of three persona reviewers runs in parallel against the phase's artifact, a synthesizer consolidates their findings, and the human gatekeeper (Robert) approves, revises, or aborts. Inside each phase, a planner-executor-verifier loop runs with a configurable cap. The orchestrator is built on Claude Code (sub-agents + a `/rep` slash command), is stack-agnostic, and mirrors per-rep state to a central reps-registry for cross-rep visibility.

## Top-level pattern

Two well-known orchestration patterns combined:

1. **Sequential pattern between phases.** Each phase's approved output is the next phase's input. Deterministic ordering. Failures surface and resolve before the next phase runs.
2. **Concurrent pattern at gates.** When a phase completes, three persona reviewers run in parallel against the artifact; `gate-synthesizer` consolidates their findings into a single packet for the human gate.

Inside each phase, a **maker-checker / planner-executor-verifier** loop runs until the verifier passes or hits the per-phase iteration cap. For lighter phases (scope, design), the loop collapses to: agent drafts → self-verify → return.

## Lifecycle phases

For each phase, the artifact is the input to the gate. Reviewers read the artifact; they never modify it.

| Phase | Primary agent(s) | Artifact | Gate reviewers |
|---|---|---|---|
| scope | `scoping-agent` | `.orchestrator/scope/scope.md` | viability-skeptic, scope-cutter, reps-fit-reviewer |
| design | `design-agent` | `.orchestrator/design/design.md` | feasibility-reviewer, simplicity-reviewer, security-data-reviewer |
| build | `build-planner` → `build-executor` → `build-verifier` (loop) | code + `.orchestrator/build/build-log.md` | correctness-reviewer, maintainability-reviewer, test-coverage-reviewer |
| test | `test-runner-agent` | `.orchestrator/test/test-report.md` | edge-case-prober, ux-smoke-reviewer, launch-blockers-reviewer |
| deploy | `deploy-target-detector` → per-target deploy specialist | `.orchestrator/deploy/deploy-manifest.md` | release-readiness-reviewer, rollback-reviewer, **retro-author (required)** |

## Iteration caps

Configurable in `config/iteration_caps.yml`. Hitting a cap **never silently fails** — it surfaces a "cap reached" gate to Robert with three explicit options: approve as-is, revise with explicit human guidance, kill the rep.

Defaults: scope 2, design 3, build 5, test 3, deploy 2.

## Gate mechanism

Pluggable interface so v2 can swap gate backends without changing phase agents.

Interface contract:

```
present_gate(phase, artifact_path, review_packet) -> { approved | revise(feedback) | abort }
```

**v1 backend (the only one implemented):** interactive in-session. The `/rep` command reads `gate-review.md` and surfaces an AskUserQuestion-style prompt with three options.

**Planned v2 backends (not yet built):**
- `GitHubPRGate` — each gate opens a PR in the rep's repo containing the artifact + review packet. Approving the PR advances the phase.
- `FilesystemStatusGate` — each gate writes the review packet to a known path and watches for `status: approved` in a status file. Useful for async workflows.

## State and artifacts

**Source of truth:** each rep's `.orchestrator/` folder, in the rep's own git repo.

```
<rep-repo>/.orchestrator/
├── status.yml
├── scope/{scope.md, reviews/*.md, gate-review.md}
├── design/{design.md, reviews/*.md, gate-review.md}
├── build/{plan-*.md, build-log.md, verifier-*.md, reviews/*.md, gate-review.md}
├── test/{test-report.md, failure-*.md, reviews/*.md, gate-review.md}
├── deploy/{target-detection.yml, deploy-manifest.md, reviews/*.md, gate-review.md}
└── retro.md
```

**Mirror:** `registry-sync-agent` pushes a one-way snapshot to `semiagenticRob/reps-registry` after every gate approval. Each sync is a separate commit:

```
sync: rep-{id} {phase} {result}
```

The registry's git log itself is the portfolio audit trail.

## scope.md fields (the input contract for everything downstream)

| Field | Type | Notes |
|---|---|---|
| Problem statement | prose | one paragraph |
| Target user / who pays | prose | who pays, how they find it |
| Success criteria | prose | observable test for "done" |
| Kill criteria | prose | signals to stop |
| `archetype_hint` | enum | `web-app` \| `mobile` \| `static-site` \| `daemon` \| `agentic` \| `cli` \| `other` |
| `effort_budget.hours_to_mvp` | number | your estimate |
| `effort_budget.hours_hard_cap` | number | when the orchestrator may surface a kill recommendation |
| Monetization hypothesis | prose | passive-income angle |
| Related prior reps | list | manual links — used for lightweight cross-rep memory |

## Deploy target detection

`deploy-target-detector` outputs a ranked candidate list, plus `selected` and a permanent `fallback: manual-checklist`:

```yaml
candidates:
  - target: vercel-next
    confidence: high
    evidence: "next.config.js present, vercel.json present"
  - target: gh-pages-static
    confidence: low
    evidence: "no server-side code, but vercel.json present"
selected: vercel-next
fallback: manual-checklist
scope_hint: web-app
design_planned_target: vercel-next
conflict: none
```

The deploy-gate reviewer `release-readiness-reviewer` cross-checks this independently, so a wrong selection gets caught before any deploy runs.

## Cross-rep memory (lightweight)

- Retros are free-form markdown at `<rep-repo>/.orchestrator/retro.md`.
- The reps-registry holds an `index.yml` summarizing every rep (title, archetype, phase, latest gate result).
- During scope, `scoping-agent` asks Robert "any related prior reps?" — manual linking only. No embedding-based retrieval in v1.

## What's explicitly out of scope (v1)

- Portfolio-level orchestration / concurrent reps
- Automated cross-rep retrieval (embeddings, similarity search)
- Real-time observability dashboard
- PR-based and file-based gate backends
- Per-target deploy specialists for archetypes not yet encountered (added incrementally)
- Automatic idea-backlog ingestion

## Verifying the orchestrator itself

Pick a small, well-understood rep idea and run it end to end. Confirm:

1. `/rep new "<idea>"` starts at scope, asks the right questions, produces a `scope.md` you'd accept on the merits.
2. The scope gate fires with a `gate-review.md` consolidating all three reviewers.
3. Each subsequent phase advances on approval and produces its artifact.
4. Build phase produces tests; test phase runs them and reports honestly.
5. Deploy-target-detector picks correctly (or fails to `manual-checklist`).
6. The reps-registry receives a clean mirror commit after each gate.
7. `/rep resume` picks up at the current phase after a session restart.
8. `revise` feedback at a gate routes back to the right phase and the iteration cap is honored.
