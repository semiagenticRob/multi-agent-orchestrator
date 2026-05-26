# Testing the orchestrator

The orchestrator's design has two distinct things to verify:

1. **The artifacts are correct** — does the scope.md a real run produces match the template, does the synthesized gate review actually consolidate the three reviews, does the registry sync produce a clean commit per gate. These are testable from any Claude Code session that role-plays the agents (no sub-agent dispatch needed). This kind of testing was used during initial dogfooding of rep-015 (HodlForWhat).

2. **The sub-agent dispatch mechanism works** — does Claude Code actually invoke the agents under `.claude/agents/` via the `Agent` tool when the `/rep` slash command runs. This requires a real session launched from the orchestrator repo. Role-play does not exercise it.

## Validating the sub-agent dispatch (manual, do this before relying on the orchestrator on a real rep)

1. **Launch a clean Claude Code session from the orchestrator repo:**
   ```bash
   cd ~/multi-agent-orchestrator
   claude
   ```
   Doing it from this directory is what makes the `/rep` slash command and the agents in `.claude/agents/` discoverable.
2. **Smoke-test a scope-phase agent dispatch.** Inside the session, ask Claude to dispatch the `scoping-agent` sub-agent directly (not via `/rep`) with a one-line idea. The Agent tool should accept `subagent_type: "scoping-agent"`. If it errors with "agent not found," the agent file's frontmatter isn't being picked up — verify `name:` matches the filename stem.
3. **Smoke-test parallel reviewer dispatch.** Have Claude dispatch `viability-skeptic`, `scope-cutter`, and `reps-fit-reviewer` in a single message with three Agent tool calls. All three should run in parallel and write their review files. If they run sequentially, the parallel-dispatch instruction isn't being respected — usually a prompt issue, not a tool issue.
4. **Smoke-test the full `/rep adopt` flow on a throwaway repo.** Make a tiny test repo (`mkdir ~/test-rep && cd ~/test-rep && git init && cd ~/multi-agent-orchestrator && claude`), then inside the session run `/rep adopt rep-test ~/test-rep`. The scoping-agent should be dispatched and run an interview. After the gate, registry-sync-agent should produce a real commit on `~/reps-registry`.
5. **Walk through to gate-synthesizer.** Confirm the synthesizer's verdict, reviewer summary table, and disagreement section read clearly. The synthesizer is the highest-leverage agent — if its output is unclear, fix its prompt first.

## What to check after each phase runs for real

- `<rep-repo>/.orchestrator/status.yml` updated to the right phase and iteration count
- `<rep-repo>/.orchestrator/<phase>/<artifact>.md` exists and follows the template
- `<rep-repo>/.orchestrator/<phase>/reviews/*.md` exists for every gate reviewer (count matches the phase)
- `<rep-repo>/.orchestrator/<phase>/gate-review.md` exists with a verdict
- `~/reps-registry/reps/<rep-id>/` mirrors the rep's `.orchestrator/`
- `~/reps-registry/index.yml` has an entry with the latest gate result
- The registry's git log shows one commit per gate, formatted `sync: rep-{id} {phase} {result}`

## Known limitations of role-play testing

When the orchestrator is exercised by role-play (Claude in a session NOT launched from `~/multi-agent-orchestrator`), the following are NOT actually tested:

- Whether the `Agent` tool routes correctly to sub-agent types matching `.claude/agents/*.md`
- Whether the parallel dispatch of three reviewers actually runs concurrently
- Whether sub-agent tool restrictions (`tools:` frontmatter) are enforced
- Whether sub-agent isolation works (each sub-agent gets a clean context window)

These need a clean-session run to confirm.
