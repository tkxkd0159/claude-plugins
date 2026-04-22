---
name: code-review
description: Adversarial, evidence-based review of HEAD against a base branch by parallel lens subagents
disable-model-invocation: true
argument-hint: "[--base BRANCH] [--comment]"
---

Adversarial code review of HEAD against a base branch. Dispatches up to 7 focused lens subagents in parallel, validates findings against the actual code, and reports them — optionally posting to the PR.

**Agent assumptions (applies to every subagent):**

- All tools work. Do not test or probe tools.
- Call a tool only when it is required to complete the task.
- Cite every finding by `file:line-line`. Never invent a location; if unsure, move the item to residual risks.
- Zero findings is a valid, sometimes correct answer.

## Rubrics

**Severity**

- `critical` — exploitable security, auth bypass, data loss, corruption, severe outage
- `high` — likely production failure or serious regression under realistic conditions
- `medium` — real bug under a plausible edge case
- `low` — non-trivial, worth fixing, not a blocker

**Confidence**

- `high` — directly supported by the cited code
- `medium` — strong evidence, one assumption remains
- `low` — plausible but speculative; prefer residual risk over a low-confidence finding

## Three-phase protocol

Every lens subagent follows this protocol. Do not merge phases; each depends on the previous one.

1. **Evidence** — walk the diff inside the assigned focus and record 10–20 observations: `file:line-line`, current vs. prior behavior, invariants/callers touched, which inputs can reach it. Mark anything not determinable from the diff as `unknown: <what to check>`. No severity, no recommendations yet.
2. **Candidates** — propose findings that cite phase-1 observations by `file:line`. Each candidate records: Title, Severity, Confidence, Evidence, Failure mode (concrete path to user harm), Recommended fix. Drop any candidate that cannot cite phase-1 evidence.
3. **Self-critique** — steelman the PR author for each candidate, then mark `KEEP` / `DOWNGRADE` (with reason) / `DROP`. Emit a `FINAL` block:
   - `Verdict`: `pass` | `concern` | `blocker`
   - `Checked`: what was actually examined
   - `Findings`: surviving findings, ordered by severity then confidence, **max 5**
   - `Residual risks`: unverified suspicions explicitly not promoted to findings

## Lens focus (used in step 3)

- **Correctness** — logic errors, null handling, off-by-one, control flow, contract mismatches, stale callers, silent data corruption, partial failures, rollback gaps, migration/schema mismatch, cache staleness, dead or unreachable code.
- **Architecture** — module boundaries, dependency direction, abstractions placed in the wrong layer, cross-cutting concerns wired ad-hoc, structural layering breaks, new cross-module coupling.
- **Security** — auth/authz, input validation, injection (SQL/command/template), secret exposure, unsafe deserialization, SSRF, path traversal, privilege boundaries, multi-tenant leakage, unsafe defaults, PII handling.
- **Maintainability** — hidden coupling, brittle abstractions, duplication, dead flags, unclear invariants, misleading names, obvious simplification not taken, accidental complexity.
- **Testing** — coverage gaps for changed behavior, assertions that don't test what they claim, time/random/async-timing flakiness, shared fixtures, order coupling, brittle mocks.
- **Performance** — DB query shape, N+1, hot-path allocations, nested loops over user data, avoidable round-trips, missing or wrong caching, sync work that should be async.
- **Concurrency** — non-atomic read-modify-write, lock ordering, cancellation hazards, idempotency gaps, visibility/ordering bugs, retry interactions, double-submit/double-process.

## Procedure

Follow these steps precisely.

1. **Parse arguments and pin the comparison point.**
   - `BASE_BRANCH`: use `$0` if non-empty, otherwise `origin/main`.
   - `MERGE_BASE=$(git merge-base "$BASE_BRANCH" HEAD)`.
   - `COMMENT=1` if `--comment` was passed; otherwise unset.
   - Every subsequent step diffs against `"$MERGE_BASE"..HEAD` so all reviewers see identical bytes.

2. **Dispatch** (haiku subagent).
   Decide which lenses to run. `correctness` always runs; the others gate on triggers below. Use `git diff --name-only`, `git diff --shortstat`, `git diff --name-only --diff-filter=A`, and `grep -E` on the diff body. Read git outputs as text.

   | Lens            | Trigger                                                                                                                                                                               |
   | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | correctness     | always                                                                                                                                                                                |
   | architecture    | ≥3 files changed, OR new top-level module/directory, OR import-graph/public-export changes                                                                                            |
   | security        | paths match `auth\|session\|token\|crypto\|secret\|endpoint\|api\|middleware`, OR diff touches input validation, deserialization, subprocess/exec, SQL construction, network I/O, PII |
   | maintainability | ≳100 added lines, OR new abstractions (class/interface/trait/struct), OR rename-heavy diff                                                                                            |
   | testing         | any `*_test.*` / `*.spec.*` / `tests/**` change, OR production change with no adjacent test                                                                                           |
   | performance     | DB query changes, loops over user data, hot-path handlers, network calls, caching/memoization changes                                                                                 |
   | concurrency     | shared mutable state, locks/mutexes, async/await, goroutines/threads, channels, queues/retries, cancellation, idempotency code                                                        |

   Triggers are heuristics. If a small diff plainly needs a lens (e.g., a two-file auth middleware rewrite), run it regardless of raw counts.

   Emit a dispatch summary: `activated` lenses + `skipped` lenses (each with a one-line reason).

3. **Launch lens subagents in parallel.**
   For every activated lens, launch one subagent with `$MERGE_BASE`, the lens's focus paragraph, the rubrics, and the three-phase protocol. Do not share a lens's focus with its neighbors — bounded scope is the point.

   Models: use **opus** for `correctness`, `security`, `concurrency` (highest-stakes lenses). Use **sonnet** for `architecture`, `maintainability`, `testing`, `performance`.

4. **Validate surviving findings in parallel.**
   For every `KEEP` or `DOWNGRADE` finding across all lenses, launch a validator subagent. Pass the finding's title, severity, `file:line-line` evidence, failure mode, and the PR diff. The validator opens the referenced range in the current code and returns either `CONFIRMED` or `REJECTED: <one-line reason>`. Use **opus** to validate `correctness` / `security` / `concurrency` findings; **sonnet** for the rest.

5. **Aggregate.**
   - Drop findings where validation returned `REJECTED`.
   - Merge duplicates across lenses: two findings are duplicates when their `file:line` ranges overlap (or fall within ~5 lines) and their failure modes match. Keep the higher severity/confidence; note the second lens as a corroborating source.

6. **Gap analysis — one pass only.**
   Look for issues that emerge only from combining lenses (e.g., a medium correctness finding plus a medium concurrency finding that together imply a high data-loss path). Do **not** re-critique individual findings; each lens's phase-3 self-critique is authoritative. Stop after this pass.

7. **Report to terminal.**
   Output in this order:
   - **Executive summary** — base branch, `$MERGE_BASE` SHA, changed-file count, activated lenses, skipped lenses (with reasons), blockers, residual risks.
   - One section per **activated** lens, with its surviving findings (Title, `SEV/CONF`, `file:line`, Failure mode, Fix). Omit skipped lenses entirely.
   - **Cross-cutting risks** — from gap analysis.
   - **Approval** — exactly one of `ready to approve` / `ready after fixes` / `not ready`. Use `ready to approve` only if no unresolved `high`/`critical` findings and no material residual risks. State explicitly what must be fixed before approval.

   If `COMMENT` is unset, stop here and present the post-review menu (below).

8. **Post to GitHub** (only if `COMMENT` is set).
   Invoke the `add-pr-review` skill with the final finding list. Do not attempt to post via `gh pr comment` directly.

## False positives — do NOT flag

- Pre-existing issues this PR neither introduces nor worsens.
- Stylistic nits, formatting, subjective preferences.
- Missing tests unless the gap creates a concrete regression risk in **this** PR.
- Anything a linter would catch.
- General "you should probably…" code-quality concerns without a concrete failure mode.
- Issues already suppressed in-code (lint-ignore comments, etc.).
- Generated files, vendored code, lockfiles, snapshots, docs-only diffs — unless they affect runtime, build, or security.
- Claims the subagent cannot verify — downgrade to residual risk instead.

## Post-review menu

If `COMMENT` was not passed, ask the user to choose one:

1. **Continue review** — go deeper on a specific finding, file, subsystem, or a skipped lens.
2. **Post all findings to GitHub** — invoke `add-pr-review` with the full list.
3. **Edit findings first** — present a compact list (`[#N] [SEV/CONF] Title — file:line`), allow removals / severity changes / rewording / fix edits, show the updated list after each edit round, then invoke `add-pr-review` with the final list.
4. **Exit** — end the session without posting anything.
