# Architecture

## Four phases

1. **Fetch** — `src/lib/github.js` pulls the PR metadata and its changed
   files (with unified diff patches) from the GitHub REST API. The offline
   demo skips this and loads `fixtures/demo-pr.js` instead — same shape,
   fixed data.
2. **Fan out** — `src/lib/runGate.js` calls all five subagents in
   `Promise.all`, passing each the identical `context` object
   (`{ number, title, body, files, specText }`). Each subagent only reads the
   fields it needs and never talks to another subagent.
3. **Aggregate** — `src/lib/aggregator.js` takes the five `{ id, status,
   score }` results, applies a fixed weight per subagent, and produces one
   `riskScore` (0–100, higher = riskier) and its complement,
   `readinessScore = 100 - riskScore`, plus one overall `pass` / `warn` /
   `block` verdict.
4. **Report** — `src/lib/report.js` prints a colored console report;
   `src/index.js` can additionally post the same findings as a single PR
   comment via `--post`.

## The five agents

| id | file | returns risk when |
|---|---|---|
| `security` | `subagents/security.js` | a hardcoded secret/key pattern or an unsafe call (`eval(`, `child_process`, `new Function(`) is added; a softer warning when `package.json` changes |
| `testCoverage` | `subagents/testCoverage.js` | a changed source file has no correspondingly-named test file changed in the same diff |
| `breakingChange` | `subagents/breakingChange.js` | a diff removes a line starting with `export function` / `export class` / `export const` / `export let` and that symbol name doesn't reappear in any added line |
| `specConformance` | `subagents/specConformance.js` | fewer than 70% of the `-` bullet requirements in the supplied spec text have at least half their keywords present in the PR title/body/diff |
| `changelog` | `subagents/changelog.js` | never — it's informational only (weight 0, always `status: "info"`) |

Every subagent returns the same shape:

```js
{ id, name, status, score, findings }
// status: "pass" | "warn" | "block" | "info"
// score:  0-100, HIGHER = MORE RISK (not a quality score)
```

## Scoring formula

```
WEIGHTS = { security: 0.35, testCoverage: 0.25, breakingChange: 0.25,
            specConformance: 0.15, changelog: 0 }

riskScore      = round( Σ(score_i * weight_i) / Σ(weight_i) )
readinessScore = 100 - riskScore

overallStatus  = "block" if any agent is "block"
                 else "warn" if any agent is "warn"
                 else "pass"
```

Security carries the largest weight because a hardcoded secret or an unsafe
call is the one class of finding that's dangerous even in isolation.
`changelog` has weight 0 so it never affects the verdict — it's always along
for the ride as a suggested entry, never a blocker.

## File structure

```
src/
  index.js               CLI: demo + single-PR gate (+ optional --post)
  background.js          CLI: scan all open PRs, sorted by risk
  lib/
    github.js            GitHub REST API client (fetch PR/files, list PRs, post comment)
    runGate.js            Fans out to all subagents in parallel, then aggregates
    aggregator.js         Weighted scoring + overall pass/warn/block verdict
    report.js             Console report formatter
    subagents/
      security.js
      testCoverage.js
      breakingChange.js
      specConformance.js
      changelog.js
fixtures/
  demo-pr.js              Sample PR payload for the offline demo
  demo-spec.md            Sample spec doc the demo PR is checked against
demo/
  root-cause-race.html    Standalone interactive visual (not part of the CLI pipeline)
submission/
  index.html              This package's launcher page
```

## Extension guide

To add a sixth subagent:

1. Create `src/lib/subagents/yourCheck.js` exporting `async function
   run(context)` that returns `{ id, name, status, score, findings }`.
2. Import it and add it to the `Promise.all` array in `src/lib/runGate.js`.
3. Give it a weight in `WEIGHTS` in `src/lib/aggregator.js`. Use `0` if it
   should be informational only, like `changelog`.
4. If it should also appear in the posted PR comment, it will automatically —
   `buildCommentBody` in `src/index.js` includes every agent except
   `changelog` by id.

No other file needs to change. The `context` object already carries
everything a new subagent is likely to need (`title`, `body`, `files` with
patches, and `specText`); extend `context` in `runGate.js` if a new subagent
needs something else.
