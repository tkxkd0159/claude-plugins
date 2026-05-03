---
name: add-pr-comments
description: Use when posting validated code review findings as GitHub PR line-level comments or a GitHub PR review after completing a review.
---

# Add PR Comments

Post validated code review findings from the current review as a GitHub PR review with line-level comments. Use this only after findings have been reviewed, deduplicated, and validated against the current diff.

## Procedure

1. **Resolve the PR.**
   - Run: `gh pr view --json number,headRefOid,url -q '{number: .number, sha: .headRefOid, url: .url}'`
   - Extract PR number, HEAD commit SHA, and PR URL.
   - If this fails because the current branch has no PR, ask the user for the PR number and repo.

2. **Confirm review language.**
   - If the user has not specified a language for the review comments, ask before posting.

3. **Use the final finding list.**
   - Use only findings from the completed review.
   - Each finding should include: title, severity, confidence, evidence (`file:line` or `file:startLine-endLine`), why it matters, and recommended fix.
   - If the user edited findings, use the edited set.

4. **Classify findings.**
   - Line-anchored: findings with evidence in `file:line` or `file:startLine-endLine` format.
   - General: findings without a specific file/line reference.

5. **Filter and deduplicate.**
   - Skip `low` confidence findings unless severity is `high` or `critical`.
   - Merge duplicates that point to the same file/line or same failure mode; keep the higher-severity framing.

6. **Build the GitHub review payload.**
   - Get owner/repo: `gh repo view --json nameWithOwner -q .nameWithOwner`
   - POST to `/repos/{owner}/{repo}/pulls/{pr_number}/reviews`.
   - Payload fields:
     - `commit_id`: HEAD commit SHA from step 1
     - `event`: `APPROVE` for `ready to approve`, otherwise `REQUEST_CHANGES`
     - `body`: one-sentence TL;DR of the PR's overall health
     - `comments`: line-anchored comments, each with:
       - `path`: relative file path from repo root
       - `line`: line number, or range end line
       - `start_line`: range start line, only for multi-line ranges
       - `side`: `RIGHT`
       - `body`: formatted finding comment

7. **Format each comment body.**

   ```markdown
   **[SEVERITY] Title**

   Why it matters.

   **Recommended fix:** actionable fix
   ```

   Use severity values `CRITICAL`, `HIGH`, `MEDIUM`, or `LOW`.

8. **Post the review.**
   - Send the payload with `gh api ... --input <payload-file>`.
   - If there are more than 50 line-anchored comments, split them into batches of 50.

9. **Handle errors.**
   - `422`: move comments that cannot be anchored to the diff into the review body, then retry.
   - `403` or `404`: report that the `gh` token or user may lack write access.
   - If the PR HEAD SHA differs from the reviewed SHA, warn the user and ask before posting.

10. **Report the result.**
    - State the number of line comments posted.
    - State the number of general findings included in the body.
    - Provide the PR link.
    - State the review event used: `APPROVE` or `REQUEST_CHANGES`.
