# Impact Metrics

## Before / after, on the demo PR

| | Manual review | Release Gate |
|---|---|---|
| Time to first finding | Minutes to hours, depending on reviewer load | ~1 second, no network (offline demo) |
| Findings surfaced | Whatever the reviewer happens to notice | 4/4 planted issues caught: hardcoded secret, missing tests, removed exported function, unmet spec requirement |
| Review comments produced | Often one comment per issue, sometimes across multiple review rounds | 1 aggregated comment via `--post` |
| Cost per PR | Reviewer time | One GitHub API call pair (PR + files); no LLM calls, no per-PR billing |

These are illustrative numbers from the bundled fixture, not a claim about
performance on arbitrary real-world PRs — see "Measurement framework" below
for how to validate that on a real repo.

## Measurement framework

To validate impact on a real codebase rather than the fixture:

1. **Retrospective run.** Point `node src/background.js <owner> <repo>` at a
   repo with a known incident history. For each past PR that later caused a
   production issue, check whether Release Gate's subagents would have
   flagged it (hardcoded secret, missing tests, breaking change, unmet spec
   requirement).
2. **False positive rate.** Run against a sample of recent merged PRs that
   shipped cleanly. Count how many were flagged `block` or `warn` and
   inspect whether the flag was justified — this bounds the "noise" cost of
   adopting it.
3. **Time-to-signal.** Compare wall-clock time from PR open to first
   actionable comment, with and without Release Gate running on PR open via
   CI.
4. **Weight tuning.** If a category (e.g. `testCoverage`) generates too many
   warnings relative to real risk, adjust its weight in `aggregator.js`
   rather than the subagent's own logic — the score/weight separation exists
   for exactly this.

## ROI estimate (rough, for a mid-size team)

Assumptions: 20 PRs/day, 10 minutes of reviewer time saved per PR when a
security/breaking-change/missing-test issue is caught before human review
instead of during it, $75/hr fully-loaded reviewer cost.

```
20 PRs/day × 10 min saved × ($75/hr ÷ 60 min) ≈ $250/day
≈ $5,000/month, ≈ $60,000/year
```

This is a directional estimate, not a benchmarked figure — the retrospective
run above is the way to get a real number for a specific team, since hit
rate and false-positive rate both vary by codebase and PR volume.

## What isn't measured yet

- Real-world false negative rate (issues that slip past all five subagents).
- Impact on PR cycle time when `--post` is wired into CI on every PR open.
- Whether the changelog draft is used as-is or edited — it's informational
  and currently unweighted, so it has no effect on the verdict either way.
