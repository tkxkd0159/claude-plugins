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

2. **Set review language.**
   - Use the language specified by the user.
   - If no language is specified, use English.

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
     - `body`: concise human review note; see body rules below
     - `comments`: line-anchored comments, each with:
       - `path`: relative file path from repo root
       - `line`: line number, or range end line
       - `start_line`: range start line, only for multi-line ranges
       - `side`: `RIGHT`
       - `body`: formatted finding comment

7. **Format the review body.**
   - Keep the body short and natural, like a human PR review note.
   - If all findings are line-anchored, write one short sentence.
   - Include brief bullets only for general findings or residual risks that cannot be expressed as line comments.
   - Do not duplicate line-comment details in the body.
   - Do not include praise, PR recaps, severity summaries, or approval/status lines such as `Approval:` or `ready after fixes`.

   Example:

   ```markdown
   Left three comments on quota exclusion edge cases and test coverage.

   Also:
   - The const naming is slightly inconsistent; consider a short godoc if you touch this area again.
   - The metric boilerplate has grown, but it follows the existing pattern.
   ```

8. **Format each line comment body.**

   ```markdown
   **[SEVERITY] Title**

   Why it matters.

   **Recommended fix:** actionable fix
   ```

   Use severity values `CRITICAL`, `HIGH`, `MEDIUM`, or `LOW`.

9. **Post the review.**
   - Send the payload with `gh api ... --input <payload-file>`.
   - If there are more than 50 line-anchored comments, split them into batches of 50.

10. **Handle errors.**
   - `422`: move only the necessary unanchored findings into terse body bullets, then retry. Do not move full line-comment bodies into the review body.
   - `403` or `404`: report that the `gh` token or user may lack write access.
   - If the PR HEAD SHA differs from the reviewed SHA, warn the user and ask before posting.

11. **Report the result.**
    - State the number of line comments posted.
    - State the number of general findings or residual risks included in the body.
    - Provide the PR link.
    - State the review event used: `APPROVE` or `REQUEST_CHANGES`.
