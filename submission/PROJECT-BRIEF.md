# Release Gate — Project Brief

**IBM Bob 2.0 Hackathon submission**

## What it is

Release Gate is a zero-dependency Node.js agent that turns a pull request into
a single, actionable readiness verdict instead of a pile of separate review
comments. Five subagents each look at one concern in isolation, run in
parallel, and hand back a short summary. An aggregator combines their scores
into one Readiness Score (0–100) and one pass / warn / block verdict.

## The problem

Reviewers either skim a big diff and miss the one dangerous line, or they read
every line and burn an hour on a PR that adds a config flag. Automated checks
that do exist (linters, CI) don't reason about the diff's *meaning* — a
hardcoded secret, a quietly removed public function, a spec requirement that
was never implemented.

## The solution

Five focused subagents, each isolated to a single concern:

| Subagent | Checks |
|---|---|
| Security & Risk | Hardcoded secrets/keys, unsafe calls (`eval`, `child_process`), dependency bumps |
| Test Coverage | Whether changed source files have matching test file changes in the same diff |
| Breaking Change Trace | Exported functions/classes/consts removed with no replacement |
| Spec Conformance | Checks the PR against a supplied spec/design doc's stated requirements |
| Changelog Draft | Drafts a changelog entry from the PR title and changed files (informational) |

Their outputs are combined by a weighted aggregator into one Readiness Score,
so a human sees one verdict and the handful of findings that drove it — not
five separate bot comments.

## Demo story

The offline demo (`node src/index.js demo`) gates a PR that looks fine at a
glance — "add retry logic to a webhook handler" — but actually:

- adds a hardcoded Stripe secret,
- removes a legacy function the spec says must stay available,
- bumps a dependency with no compatibility note, and
- touches no test files.

Release Gate catches all four in one pass and returns a 12/100 readiness
score with a BLOCK verdict, each with the exact line or requirement that
triggered it.

## Bob 2.0 feature map

- **Isolated subagents, parallel execution** — each subagent gets the same
  context but reasons about only its own concern, mirroring Bob 2.0's
  subagent model.
- **Summary over noise** — each subagent returns `{ status, score, findings }`,
  not a raw transcript.
- **Single aggregated output** — one Readiness Score and one report instead of
  N separate messages.
- **Extensible by design** — a new subagent is a file with an async `run()`
  and a weight in `aggregator.js`; see `ARCHITECTURE.md`.

## Try it

```
node src/index.js demo
```

No token, no network, no `npm install`. See `SETUP-GUIDE.md` for gating a
real PR.
