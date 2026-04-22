---
name: add-pr-comments
description: Use when posting code review findings as GitHub PR line-level comments after completing a review
---

Post review findings from the current conversation as a GitHub PR review with line-level comments.
This skill is invoked from subagents-review after the user chooses to post findings.

Process

1. Resolve the PR:
   - Run: `gh pr view --json number,headRefOid,url -q '{number: .number, sha: .headRefOid, url: .url}'`
   - Extract PR number, HEAD commit SHA, and PR URL
   - If this fails (no PR for current branch), ask the user for the PR number and repo

2. Extract findings from conversation context:
   - Parse all findings produced by the preceding review
   - Each finding has: title, severity, confidence, evidence (file:line), why it matters, recommended fix
   - If the user edited findings (option 3 in subagents-review), use the edited set

3. Classify findings into two groups:
   - Line-anchored: have evidence in `file:line` or `file:startLine-endLine` format
   - General: no specific file:line reference

4. Filter and deduplicate:
   - Skip findings with confidence `low` unless severity is `high` or `critical`
   - Merge duplicates that point to the same file:line — keep the higher-severity framing

5. Build the review JSON payload for `POST /repos/{owner}/{repo}/pulls/{pr_number}/reviews`:
   - `commit_id`: HEAD commit SHA from step 1
   - `event`: map the final approval assessment:
     - `ready to approve` -> `"APPROVE"`
     - `ready after fixes` -> `"REQUEST_CHANGES"`
     - `not ready` -> `"REQUEST_CHANGES"`
   - `body`: executive summary from the review, plus any general findings (not line-anchored) formatted as a bulleted list
   - `comments`: array of line-anchored comment objects, each with:
     - `path`: relative file path (from repo root)
     - `line`: line number (for ranges, use the end line)
     - `start_line`: start line (only for multi-line ranges; omit for single-line)
     - `side`: `"RIGHT"`
     - `body`: formatted comment (see step 6)

6. Format each comment body:

   ```
   **[SEVERITY] Title** (confidence: X)

   Why it matters.

   **Recommended fix:** actionable fix
   ```

   Where SEVERITY is one of: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`.

7. Post the review:
   - Get owner/repo: `gh repo view --json nameWithOwner -q .nameWithOwner`
   - Write the JSON payload to a temp file or pipe via stdin
   - Run: `echo '$JSON' | gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --input -`
   - If there are more than 50 line-anchored comments, split into batches of 50 and post multiple reviews (GitHub API limit)

8. Handle errors:
   - 422 (Unprocessable Entity) for comments on lines not in the diff: move those comments to the review body as general findings, then retry
   - 403/404: inform the user their `gh` token may lack write access to the repo
   - If HEAD SHA from step 1 differs from what was reviewed (force-push occurred), warn the user before posting

9. Report the result:
   - Print: number of line comments posted, number of general findings in body
   - Print: link to the PR
   - Print: the review event type used (APPROVE / REQUEST_CHANGES)

You MUST ask in which language they would like the review to be written before leaving it.
