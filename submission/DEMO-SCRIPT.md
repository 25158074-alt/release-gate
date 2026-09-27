# Demo Script — 4 minutes for judges

## 0:00 – 0:30 — The problem (say this, no screen action needed)

"Reviewers either skim a diff and miss the dangerous line, or read every line
and burn an hour on a trivial PR. Automated checks like linters don't reason
about what a diff *means* — a hardcoded secret, a quietly removed public
function, a spec requirement nobody implemented. Release Gate is five small
agents that each watch one of those things, and one aggregator that turns
their findings into a single readiness verdict."

## 0:30 – 1:00 — Show the code, not slides

Open `src/lib/runGate.js` and `src/lib/subagents/security.js` side by side.
Point out:

- Each subagent is a plain async function: same input, one concern, one
  summary out.
- They run with `Promise.all` — genuinely parallel, genuinely isolated.
- Zero dependencies — built-in `fetch` and `fs` only.

## 1:00 – 2:30 — Run the offline demo

```
node src/index.js demo
```

While it prints, narrate what's on screen in this order:

1. **Readiness Score: 12/100 — BLOCK.** "One number, one verdict, up top."
2. **Security & Risk — BLOCK.** "It caught a hardcoded Stripe secret and
   flagged the dependency bump — both real, both in the diff."
3. **Test Coverage — BLOCK.** "The changed handler has no matching test
   file changed in this PR."
4. **Breaking Change Trace — BLOCK.** "It removed `verifyWebhookLegacy` —
   an exported function — with nothing replacing it in the diff."
5. **Spec Conformance — BLOCK.** "Given `fixtures/demo-spec.md`, it checked
   the PR against each stated requirement and flagged the ones this PR
   doesn't actually satisfy — including that the legacy function must stay
   for backward compatibility."
6. **Suggested changelog entry.** "Informational only — it never affects the
   verdict, it's just there so nobody has to write it by hand."

Then: "None of this needed a model call — it's pattern-based on purpose, so
it's fast, deterministic, and free to run on every PR."

## 2:30 – 3:15 — Show the visual dashboard

Open `demo/root-cause-race.html` in a browser. Drag to orbit, click **Run
Investigation**. As the hypotheses converge:

"This is the same idea in a different shape — several hypotheses
investigated in parallel, racing toward a confidence score, until one is
confirmed. It's a standalone visual, not wired to live PR data, but it's the
same subagent-race mental model Release Gate uses underneath."

## 3:15 – 3:45 — Real PR + posting a comment

If you have a token and a spare public PR handy:

```
node src/index.js gate <pr> <owner> <repo> --spec ./spec.md --post
```

"Same pipeline, real GitHub data, and the aggregated report goes back as one
PR comment instead of five separate bot comments."

If you don't have one ready, skip this and say so — the offline demo already
proved the pipeline works end to end.

## 3:45 – 4:00 — Close

"Zero dependencies, runs in CI or locally, extensible with a new file and one
line in the aggregator. That's Release Gate."

## Anticipated Q&A

- **"Does this use an LLM?"** No — every subagent is deterministic
  pattern-matching on the diff and spec text. That's a deliberate choice: no
  API cost, no latency, no nondeterminism, and it still catches the classes
  of issue reviewers most often miss. It would be straightforward to add an
  LLM-backed subagent (e.g. subtler spec-conformance judgment) using the same
  `{ id, name, status, score, findings }` contract.
- **"What if a subagent throws?"** Currently a rejected promise in
  `Promise.all` fails the whole gate — a reasonable next step is
  `Promise.allSettled` with a subagent reporting `status: "warn"` on its own
  failure instead of aborting the run.
- **"How do you pick the weights?"** They're fixed constants in
  `aggregator.js`, chosen by how dangerous a false negative in that category
  is — security highest, changelog zero. They're easy to tune per-repo.
