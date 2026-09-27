# Setup Guide

Release Gate is zero-dependency: it only uses Node's built-in `fetch` and
`fs`. There is nothing to `npm install` to run the offline demo.

## 1. Install Node.js

You need Node 18 or later (for built-in `fetch`).

```
node --version   # should print v18.x or higher
```

If you need to install or upgrade Node, get it from https://nodejs.org.

## 2. Run the offline demo

From the project root:

```
node src/index.js demo
```

This gates a fixture PR (`fixtures/demo-pr.js`) against a fixture spec
(`fixtures/demo-spec.md`), prints a colored report to the console, and writes
the full result to `report.json`. No token and no network access are
required.

## 3. Gate a real PR

```
node src/index.js gate <prNumber> <owner> <repo>
```

Example:

```
node src/index.js gate 42 my-org my-repo
```

This works without a token for public repositories, subject to GitHub's
unauthenticated rate limit (60 requests/hour per IP).

### Optional: check against a spec file

```
node src/index.js gate <prNumber> <owner> <repo> --spec ./path/to/spec.md
```

`--spec` points at a local Markdown or text file. Every line that starts with
`-` is treated as a requirement to check the PR's title, body, and diff
against.

### Optional: post the report as a PR comment

```
node src/index.js gate <prNumber> <owner> <repo> --spec ./spec.md --post
```

Posting requires a `GITHUB_TOKEN` with `repo` write scope:

```
export GITHUB_TOKEN=ghp_your_token_here
node src/index.js gate 42 my-org my-repo --post
```

A token is also required for private repositories (with or without
`--post`).

## 4. Scan every open PR in a repo

```
node src/background.js <owner> <repo>
```

Lists every open PR, gates each one, and prints a table sorted by risk —
lowest readiness score (highest risk) first. Set `GITHUB_TOKEN` first if the
repo is private or you're hitting rate limits.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to fetch PR #N: 403 Forbidden` | Unauthenticated rate limit hit | Set `GITHUB_TOKEN` |
| `Failed to fetch PR #N: 404 Not Found` | Wrong owner/repo/PR number, or private repo without a token | Double-check the args; set `GITHUB_TOKEN` for private repos |
| `Posting a comment requires GITHUB_TOKEN with repo write scope.` | Used `--post` without a token | Set `GITHUB_TOKEN` |
| `node: command not found` | Node.js isn't installed | Install Node 18+ from nodejs.org |
