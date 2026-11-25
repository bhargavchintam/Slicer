---
name: oss-issue-contribution
description: Use when asked to pick up, fix, and open pull requests for open issues in an upstream open-source repository (GitHub), especially "take N issues", "find hard bugs I can fix", "raise a PR and watch CI until green", or when contributing from a fork to a project with its own contribution guidelines, commit-message rules, and CI.
---

# OSS Issue Contribution

## Overview

Ship small, verified, guideline-conformant fixes to upstream projects, one issue per
branch and per PR, and drive each PR until it is merged or closed.

**Core principle:** a maintainer's time is the scarce resource. Every PR must be
unclaimed, reproducible, minimal, tested, and written in the project's own voice.

## Phase 1 — Select issues (filter, then rank)

Hard filters. Drop the issue if ANY is true:

| Check | How |
|---|---|
| Assigned | `search/issues` query with `no:assignee` |
| Open or draft PR references it | list open PRs once (`is:pr is:open`, per_page=100), grep titles+bodies for the number AND for key symbols (function / file names) |
| Recently closed PR already fixed it | `is:pr is:closed <number>` and `is:pr <symbol>` |
| Someone said "I'm working on this" | read the issue comments (web page if the API is blocked) |
| A related open PR touches the same function/file | search open PRs by key symbols too; fetch its head with `git fetch upstream pull/<N>/head:pr-<N>` and diff only the author's commits; read its review comments (a diagnosis of your bug may live there; credit it) |
| Needs a maintainer decision not yet made (policy, API break, design) | comments/labels; pick only the non-controversial slice, say so |
| Cannot be verified in your environment at all | e.g. needs proprietary hardware, a 6-hour superbuild with no shortcut |

Rank survivors by: (1) match to the contributor's expertise (state the mapping
explicitly), (2) severity/impact, (3) verifiability of a fix locally, (4) small diff.
Prefer root-cause bugs over cosmetic ones; skip "unused variable" style issues unless
asked for quick wins.

Search API has a low secondary rate limit: batch into one query, write results to a
scratch file, and sleep between calls. Never loop 10+ searches back to back.

## Phase 2 — Research and fix

1. Read CONTRIBUTING, the developer guide, PR template, `.pre-commit-config.yaml`,
   CI workflows (`.github/workflows/*`), and any AI-use policy issue/doc. Write down the
   commit-message format and lint tools; they are part of the spec.
2. Reproduce first. If the full product cannot be built, build the smallest faithful
   harness: the exact function copied/compiled against the same library, a CMake
   `-P` script, a tiny ELF binary, a mocked Python module. State the harness's limits.
3. Find the root cause (`git log -S`, `git blame`) and the introducing commit.
4. Fix minimally in the project's style. Add/extend a regression test in the
   project's existing test framework, mirroring a neighbouring test.
5. Show the test fails before the fix and passes after (TDD). Keep the output.
6. When a harness check fails for BOTH the old and new code, suspect the harness first
   (missing event-loop settling, wrong precondition) and isolate it with a minimal control.
7. Stress the fix beyond the reported case (random permutations, both supported
   toolchain/library major versions) when the fix is about ordering, lifetimes or concurrency.

## Phase 3 — Local checks, then PR

- Branch per issue from latest upstream default branch, never from another fix branch.
- Run the project's linters/formatters on changed files (e.g. `pre-commit run --files ...`).
- Commit message in the project's format (e.g. Slicer: `BUG:`/`ENH:`/`COMP:` prefix,
  imperative, <72 chars, body wrapped at 80 with what/why/how, `Fixes #N`).
  Add the required co-author/attribution trailers.
- Push to the fork, open the PR against upstream's default branch, fill the PR
  template headings if one exists. Disclose AI assistance if the project asks for it.
- Never claim "fixed" or "tests pass" without command output you just ran.

## Phase 4 — Drive CI to green

- Subscribe to PR activity; do not poll with sleep.
- On failure: fetch the failing job log, classify **caused by the change** vs
  **pre-existing/flaky/infra** (check if `main` fails the same job). Fix only the former
  with a new commit; for the latter, comment with evidence.
- Fork PRs often need "Approve and run workflows" by a maintainer: if checks sit at
  `action_required`, post ONE polite comment asking a maintainer to approve the runs.
- When all required checks pass, post one short comment: what CI ran, that it is
  green, and that the PR is ready for review.
- Address review feedback with technical rigor (verify before agreeing). Push new
  commits; never force-push others' branches. Stop when the PR is merged or closed.

## Edge cases

| Situation | Do |
|---|---|
| Session/API cannot reach upstream (scoped token, wrong repo attached) | Push branches to the fork, hand the user the compare URL + ready PR title/body, say exactly what was blocked |
| Fork's default branch is stale | branch from `upstream/<default>`, not `origin/<default>` |
| Issue proposes several options | implement the least-invasive one, list the others in the PR body |
| Fix needs a dependency/toolchain you cannot build | verify with an isolated harness, mark the PR as needing maintainer CI, say so plainly |
| Another PR appears mid-work | stop, comment/compare, do not open a duplicate |
| Upstream has an AI policy | follow it; otherwise add a brief disclosure line |
| Maintainer asks for changes you believe are wrong | reply with evidence, not compliance theatre |
| Branch push restrictions in the agent environment | ask before pushing to branches other than the ones allowed |
| Commit author identity | use the configured `git config user.*` or the contributor's stated identity; never copy a name/email out of `git log` |
| Root cause is in a dependency (VTK, CTK, ITK, a vendored fork) | reproduce against the dependency alone, prepare the patch for the dependency's repo/fork, comment on the issue with the minimal repro; do not paper over it in the main repo |
| Fork and upstream share a repo name and the agent can attach only one | work on the fork; upstream PR/comment steps go to the user as a ready-to-paste handoff |
| Search API returns 403 secondary rate limit | stop looping; fetch one big page to a file and grep locally |
| Shallow clone makes `merge-base` / PR diffs huge | diff only the PR author's commits (`git log --author`), or `git fetch --deepen` |
| Two of your PRs edit the same test file | put each test in its own section/function so they merge independently; verify with a three-way merge |
| A force-push is refused by the environment | add a follow-up commit instead and offer to squash in the PR body |

## Red flags — stop

- "Nobody seems to be on it" without having grepped open PRs.
- Opening a PR whose test you never saw fail.
- Bundling two issues in one PR, or reformatting unrelated code.
- A comment that says CI passed while any required check is pending or red.
- Writing in your own voice instead of the project's (emoji, marketing tone, long essays).
