# Reviewer Output Contract

**Not an agent — a shared spec.** Every gate reviewer (viability-skeptic, simplicity-reviewer, correctness-reviewer, etc.) writes a single file at `.orchestrator/<phase>/reviews/<reviewer-name>.md` with this exact structure. `gate-synthesizer` depends on it.

```markdown
# Review — <reviewer-name>

## Verdict
pass | concerns | blocker

## Findings
- [info|concern|blocker] <one-sentence finding>. Evidence: <file:line or specific quote from artifact>.
- ...

## Recommendation
approve | revise: <one paragraph naming what specifically needs to change> | abort: <reason>
```

## Rules every reviewer follows

- Every finding carries a severity (info / concern / blocker) and concrete evidence. No "this could be better" without a citation.
- Verdict is determined by your worst finding: any blocker → `blocker`; any concern but no blocker → `concerns`; otherwise → `pass`.
- Recommendation is a single line. The synthesizer aggregates across reviewers; you don't try to balance other reviewers' opinions.
- Stay in your lane. If you notice an issue that's another reviewer's beat, mention it as `info` and move on — don't try to do their job.
- Never modify the artifact under review. You write only your own review file.
