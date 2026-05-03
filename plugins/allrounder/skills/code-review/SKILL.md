---
name: code-review
description: Use when reviewing branch or PR changes against a base branch for correctness, security, concurrency, architecture, maintainability, testing, and performance risks.
argument-hint: '[--base BRANCH] [--comment LANGUAGE]'
disable-model-invocation: true
---

# Code Review

Review the current branch or PR against a base branch with adversarial, evidence-based scrutiny. Prefer concrete regressions over style feedback.

Dispatching fresh subagent per activated review lens. We delegate tasks to specialized review agents with isolated context.

## Inputs

- Default base branch: `origin/main`
- `--base BRANCH`: compare against `BRANCH`
- `--comment [LANGUAGE]`: after the review, post only final validated findings as PR comments using `add-pr-comments` if available. Use `LANGUAGE`, or English if omitted. Without `--comment`, do not ask about or post PR comments.

## Workflow

1. Pin the comparison. Set `BASE_BRANCH` from `--base` or use `origin/main`, then:

   ```bash
   HEAD_SHA=$(git rev-parse HEAD)
   MERGE_BASE=$(git merge-base "$BASE_BRANCH" "$HEAD_SHA")
   ```

   Review `"$MERGE_BASE".."$HEAD_SHA"` for the entire pass.

2. Inventory the diff: changed files, shortstat, added/deleted files, and diff body. Ignore generated files, vendored code, lockfiles, snapshots, and docs-only changes unless they affect runtime, build, or security.

3. Choose lenses. Run `correctness` always; run other lenses when the trigger matches. Record activated and skipped lenses with one-line reasons.

   | Lens              | Trigger                                                                                                           | Review for                                                                                                         |
   | ----------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
   | `correctness`     | Always                                                                                                            | logic, null handling, contracts, stale callers, data corruption, migrations, cache staleness, unreachable code     |
   | `security`        | Auth, tokens, secrets, validation, deserialization, exec/subprocess, SQL, network I/O, endpoints, middleware, PII | auth/authz, injection, unsafe deserialization, SSRF, path traversal, tenant or PII leaks                           |
   | `concurrency`     | Shared state, locks, async, threads, queues, retries, cancellation, idempotency                                   | non-atomic updates, lock ordering, cancellation hazards, retry interactions, double-submit or double-process paths |
   | `architecture`    | 3+ files changed, new modules, public exports, import-graph or boundary changes                                   | boundary breaks, dependency direction, wrong-layer abstractions, cross-module coupling                             |
   | `maintainability` | 100+ added lines, new abstractions, duplication, hidden coupling, rename-heavy diffs                              | brittle abstractions, unclear invariants, accidental complexity, obvious simplification not taken                  |
   | `testing`         | Test changes, production changes without adjacent tests, brittle mocks, flaky assertions                          | coverage gaps for changed behavior, weak assertions, flakiness, fixture pollution                                  |
   | `performance`     | DB queries, loops over user data, hot paths, network calls, caching, avoidable round trips                        | query shape, N+1s, hot-path allocations, nested loops over user data, missing or wrong caching                     |

4. Review each activated lens:
   - Gather evidence first: cited code ranges, prior vs current behavior, touched callers, reachable inputs, unknowns.
   - Create findings only from evidence. Include title, severity, confidence, file/line, failure mode, and fix.
   - Self-critique each finding from the PR author's perspective; keep, downgrade, or drop it.

5. Validate every surviving finding directly in the current code and reviewed diff. Drop anything not supported by the cited range. Move plausible but unverified concerns to residual risks.

6. Aggregate duplicate or overlapping findings, keep the strongest severity/confidence, and run one cross-lens check for combined risks.

7. Report in this order:
   - Executive summary: base, merge-base SHA, changed-file count, activated/skipped lenses, blockers, residual risks
   - Findings by lens: title, `SEV/CONF`, `file:line`, failure mode, fix
   - Cross-cutting risks, if any
   - Approval: exactly one of `ready to approve`, `ready after fixes`, or `not ready`

## Severity

- `critical`: exploitable security issue, auth bypass, data loss, corruption, severe outage
- `high`: likely production failure or serious regression
- `medium`: real bug under a plausible edge case
- `low`: non-trivial issue worth fixing, not a blocker

## Confidence

- `high`: directly supported by cited code
- `medium`: strong evidence with one assumption
- `low`: plausible but speculative; prefer residual risk over a low-confidence finding

## Avoid

- Pre-existing issues this change neither introduces nor worsens
- Style nits, formatting, subjective preferences, or linter-level issues
- Missing tests without a concrete regression risk
- General advice without a failure mode
- Claims that cannot be verified from the diff and current code
