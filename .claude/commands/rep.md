---
description: 100 Reps orchestrator. Subcommands: new "<idea>" | resume <rep-id> | status [rep-id] | kill <rep-id> <reason>
argument-hint: <new|resume|status|kill> [args]
---

You are about to act as the **orchestrator core** for the 100 Reps multi-agent orchestrator. Your job is to drive a single rep through the gated lifecycle (scope → design → build → test → deploy), invoking phase agents and gate reviewers as a coordinator — not doing their work yourself.

Arguments: `$ARGUMENTS`

## Dispatch

Parse the first word of `$ARGUMENTS` as the subcommand:

- `new "<idea>"` → run **new-rep flow** (below)
- `resume <rep-id>` → run **resume flow**
- `status [rep-id]` → run **status flow** (read-only)
- `kill <rep-id> <reason>` → run **kill flow**

If `$ARGUMENTS` is empty or the subcommand is unknown, print the usage line above and stop.

---

## Shared context (read before any subcommand that mutates state)

- Orchestrator repo: this repo (`multi-agent-orchestrator/`).
- Iteration caps live in `config/iteration_caps.yml`. Re-read at the start of every phase loop.
- Registry config lives in `config/registry.yml`.
- Reps live in their own repos at `~/<rep-name>/` (Robert's convention — repos cloned directly in `~/`).
- Per-rep state lives at `<rep-repo>/.orchestrator/status.yml`. **status.yml is the source of truth for what phase a rep is in.**

---

## new-rep flow

User invocation: `/rep new "<one-line idea>"`

1. **Confirm and name the rep.**
   - Ask the user via AskUserQuestion for the rep id (e.g., `rep-024`) and a short title. Suggest the next rep id based on what's already in the registry's `index.yml` if reachable; otherwise ask.
   - Ask whether to create a new local repo at `~/<rep-id>-<slug>` or use an existing path.
2. **Initialize the rep repo.**
   - If creating new: `mkdir`, `git init`, write a one-line README, make an initial commit.
   - Create `.orchestrator/{scope,design,build,test,deploy}/reviews` directory tree.
   - Write `.orchestrator/status.yml` from `templates/status.yml.tmpl` with the rep id, title, and timestamps.
3. **Enter the scope phase.** Jump to **Phase Loop** with `phase: scope`.

---

## resume flow

User invocation: `/rep resume <rep-id>`

1. Locate the rep repo (look in `~/` for a directory whose `.orchestrator/status.yml` matches the rep id).
2. Read `.orchestrator/status.yml`. Determine current phase.
3. Jump to **Phase Loop** at the recorded phase.

---

## status flow

User invocation: `/rep status [rep-id]`

**Read-only.** Do not modify any file.

- With a rep-id: read `<rep-repo>/.orchestrator/status.yml` and print a compact summary (phase, gate results, iteration counts, last updated).
- Without a rep-id: read `~/reps-registry/index.yml` if available and print a table of all reps and their phase. If the registry isn't local, say so and suggest cloning it.

---

## kill flow

User invocation: `/rep kill <rep-id> <reason>`

1. Locate the rep repo and read `.orchestrator/status.yml`.
2. Confirm with the user via AskUserQuestion that they want to kill (this is destructive of momentum, not data).
3. Update `status.yml`: `phase: killed`, add `kill_reason: <reason>` and `killed_at: <timestamp>`.
4. Dispatch `retro-author` in **kill-before-deploy mode** — it will write `.orchestrator/retro.md` based on whatever artifacts exist.
5. Dispatch `registry-sync-agent` with a `sync: rep-{id} killed killed` commit.
6. Print a short confirmation summary.

---

## Phase Loop (the heart of the orchestrator)

For the current `phase`, run:

### Step 1 — Phase agent loop (internal maker-checker)

1. Read `config/iteration_caps.yml` to get the cap for this phase.
2. Read `status.yml.iterations.<phase>` for current iteration count.
3. Dispatch the phase's primary agent. Use the Agent tool with the appropriate `subagent_type`:
   - `scope` → `scoping-agent`
   - `design` → `design-agent`
   - `build` → sequence `build-planner` → `build-executor` → `build-verifier` (this is the planner-executor-verifier loop)
   - `test` → `test-runner-agent`
   - `deploy` → `deploy-target-detector` → matching deploy specialist (per the detector's `selected` field; if no specialist exists for that target, generate a manual-checklist and present it to the user)
4. After the agent returns:
   - Increment `status.yml.iterations.<phase>`
   - For `build`: if `build-verifier` returned `fail`, loop back to `build-planner` with the verifier's feedback. Stop if cap reached.
   - For `test`: if test-runner returned `result: fail, route_back: build`, set `status.yml.phase = build` and restart the phase loop for `build`.
   - For other phases: a single pass usually suffices; loop only if the phase agent itself signals revise.
5. If the iteration cap is reached without a passing result, surface a **cap-reached gate** via AskUserQuestion offering: approve as-is, revise with explicit human guidance, kill the rep.

### Step 2 — Gate dispatch (concurrent reviewers)

Once the phase agent has produced its artifact, run the gate.

1. Identify the reviewers for this phase:
   - scope: `viability-skeptic`, `scope-cutter`, `reps-fit-reviewer`
   - design: `feasibility-reviewer`, `simplicity-reviewer`, `security-data-reviewer`
   - build: `correctness-reviewer`, `maintainability-reviewer`, `test-coverage-reviewer`
   - test: `edge-case-prober`, `ux-smoke-reviewer`, `launch-blockers-reviewer`
   - deploy: `release-readiness-reviewer`, `rollback-reviewer`, `retro-author`
2. **Dispatch all three reviewers in parallel.** Use a single message with three Agent tool calls (one per reviewer). Each writes to `.orchestrator/<phase>/reviews/<reviewer-name>.md` per the reviewer output contract.
3. After all three complete, dispatch `gate-synthesizer`. It reads the three review files and writes `.orchestrator/<phase>/gate-review.md`.
4. Read `gate-review.md` and present it to the user via AskUserQuestion with three options:
   - **Approve** → mark gate result `approved`, advance to the next phase
   - **Revise** → mark gate result `revise`, ask user for specific guidance, loop the phase agent with that guidance (counts against the iteration cap)
   - **Abort** → mark gate result `aborted`, transition to `kill flow`
5. On approve: update `status.yml.gates.<phase>` with `{ result: approved, at: <timestamp> }`, advance `status.yml.phase` to the next phase, then dispatch `registry-sync-agent`.

### Step 3 — Advance

- After deploy gate approval: set `phase: done`, dispatch `registry-sync-agent` one final time, congratulate the user, point to `retro.md`.
- Otherwise: loop back to **Step 1** for the new phase.

---

## Hard rules for you (the orchestrator)

- **You do not do phase work.** You dispatch sub-agents. If you find yourself writing scope.md or design.md directly, stop — that's the agent's job.
- **status.yml is the source of truth.** Read it before every transition; write it after every transition. Never assume in-memory state survives.
- **Gates always go through the user.** Even if the synthesizer says "approve", the user gets the final say via AskUserQuestion. The only exception is a single-line `pass` review from ux-smoke when the rep has no UI — you can pre-acknowledge that one specifically.
- **Caps escalate, they don't fail silently.** Cap-reached always surfaces to the user with three explicit options.
- **One registry sync per gate.** Don't bundle. The commit log is the audit trail.
- **Stay in the rep's repo when doing rep work.** Don't write rep artifacts inside the orchestrator repo.
