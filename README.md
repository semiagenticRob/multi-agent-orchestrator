# multi-agent-orchestrator

Rep 023 — Multi-Agent Orchestrator. A meta-tool that drives a single 100 Reps project through a gated lifecycle:

```
scope  →  design  →  build  →  test  →  deploy
   │         │         │        │        │
   └───── HITL gate at each transition (concurrent persona reviewers + your go/no-go)
```

Built on Claude Code (sub-agents, slash commands). Stack-agnostic — the orchestrator itself doesn't care what the rep is built with.

## Install

This repo is the orchestrator. Two ways to use it:

**Inside Claude Code (interactive):**

```bash
git clone https://github.com/semiagenticRob/multi-agent-orchestrator.git
cd multi-agent-orchestrator
claude                       # start a session in this directory
# then inside the session:
/rep new "your idea here"
```

The `.claude/` directory (commands and sub-agents) gets picked up automatically when you launch `claude` from this repo.

**From the terminal (nanoclaw / scripts):**

```bash
# add bin/ to PATH
echo 'export PATH="$HOME/multi-agent-orchestrator/bin:$PATH"' >> ~/.zshrc
exec zsh

rep new "your idea here"
rep status
```

`bin/rep` forwards to `claude -p` by default. If you set `NANOCLAW_BIN` to your nanoclaw executable, it routes through that instead.

## Usage

```
/rep new "<one-line idea>"     start a new rep at the scope phase
/rep resume <rep-id>           resume an in-flight rep at its current phase
/rep status [rep-id]           read-only summary: one rep, or all reps from the registry
/rep kill <rep-id> <reason>    mark the rep as killed; writes a retro automatically
```

Each rep lives in its own GitHub repo (your usual convention). The orchestrator writes its per-rep state to `<rep-repo>/.orchestrator/`:

```
<rep-repo>/.orchestrator/
├── status.yml              # current phase, gate results, iteration counts
├── scope/scope.md
├── design/design.md
├── build/build-log.md
├── test/test-report.md
├── deploy/deploy-manifest.md
└── retro.md
```

After every approved gate, a one-way mirror snapshot is pushed to a central reps-registry repo (see `config/registry.yml`). The registry's git log is itself the portfolio audit trail.

## What it does (one page)

- Five phases (scope, design, build, test, deploy), each owned by a focused sub-agent.
- Internal phase pattern: planner → executor → verifier with an iteration cap (configurable per phase in `config/iteration_caps.yml`). When the cap is hit, you decide — caps never silently fail.
- At every gate, three persona reviewers run **in parallel** against the phase's artifact. A `gate-synthesizer` consolidates their reviews into one packet you read in ~60 seconds.
- You always have the final say at each gate. Approve, revise (with explicit guidance), or abort.
- Test phase routes failures back to build (within its iteration cap). Build phase loops planner → executor → verifier internally.
- Deploy phase auto-detects the target archetype and dispatches the matching deploy specialist, with `manual-checklist` as a permanent fallback.

## What's in the box

```
.claude/
├── commands/
│   └── rep.md                          /rep slash command (orchestrator core)
└── agents/
    ├── _reviewer-output-contract.md    shared spec all reviewers conform to
    ├── scoping-agent.md                ─┐
    ├── design-agent.md                  │ phase agents
    ├── build-planner.md                 │
    ├── build-executor.md                │
    ├── build-verifier.md                │
    ├── test-runner-agent.md             │
    ├── deploy-target-detector.md       ─┘
    ├── viability-skeptic.md            ─┐
    ├── scope-cutter.md                  │
    ├── reps-fit-reviewer.md             │
    ├── feasibility-reviewer.md          │ 15 gate
    ├── simplicity-reviewer.md           │ reviewer
    ├── security-data-reviewer.md        │ personas
    ├── correctness-reviewer.md          │ (3 per phase)
    ├── maintainability-reviewer.md      │
    ├── test-coverage-reviewer.md        │
    ├── edge-case-prober.md              │
    ├── ux-smoke-reviewer.md             │
    ├── launch-blockers-reviewer.md      │
    ├── release-readiness-reviewer.md    │
    ├── rollback-reviewer.md             │
    ├── retro-author.md                 ─┘ (required at deploy gate)
    ├── gate-synthesizer.md              consolidates reviews per gate
    └── registry-sync-agent.md           one-way mirror to reps-registry
config/
├── iteration_caps.yml                  per-phase loop caps
└── registry.yml                        central registry repo + sync rules
templates/
└── *.md.tmpl                           per-phase artifact templates
bin/
└── rep                                 CLI entry point (nanoclaw-aware)
docs/
└── architecture.md                     full design doc
```

## Status

v1, minimum-usable. The interactive-gate backend works; the planned PR-based and file-based gate backends are not built yet (the gate mechanism is designed to swap them in without touching phase agents). The deploy phase ships with `deploy-target-detector` and `manual-checklist` as the universal fallback — per-target deploy specialists (vercel-deployer, expo-eas-deployer, etc.) get added as you run reps of each archetype through the orchestrator.

See `docs/architecture.md` for the full design.

## License

MIT (TBD on first non-author contribution).
