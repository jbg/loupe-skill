# Remediation protocol

## Accept a finding for work

Require a canonical confirmed issue, actual severity, confidence, current-code evidence, expected invariant, and a reproducible test or explicit proof gap. Recheck related audit issues so one root-cause fix covers the correct cluster.

Classify remediation readiness before creating a worktree. Routine, focused corrections to an established invariant may proceed. If the smallest adequate remediation requires a substantial new feature, a user-visible mode or selector, a new product-policy or compatibility contract, cross-component architecture, or a broad refactor, keep the issue confirmed but put it in `Human review`. Record one explicit question describing the proposed scope and acceptance criteria, and do not open a remediation PR until the user answers it. The size or design gate is independent of severity and public-disclosure sensitivity; combine the questions only when the requested decision is unambiguous.

Determine the visibility of the audit and target repositories. For P0/P1, keep non-public evidence in the least exposed authorized location and obtain the required user decision before publishing exploit-enabling details to a more public target repository.

## Isolate the change

1. Resolve and refresh the target repository's default branch without disturbing the user's worktree.
2. Create a dedicated worktree from the current remote default branch.
3. Create a short security branch that follows the target's contribution rules and the host's required branch-prefix policy.
4. Assign exactly one writing agent to the worktree.
5. Preserve and avoid unrelated changes everywhere else.

Do not reuse a dirty worktree, force-reset user work, or combine unrelated findings.

## Validate before fixing

Trace the current source, control, sink, and boundary again. Prefer adapting the report's proposed test after reviewing it for safety. Confirm that it demonstrates the claimed invariant rather than merely exercising adjacent behavior.

If the finding does not reproduce, return it to triage with the exact counterevidence. Do not manufacture a fix to satisfy the report.

## Implement

- Make the smallest correction that enforces the invariant at the correct boundary.
- Add focused regression coverage in the repository's preferred test location.
- Include legitimate-behavior coverage.
- Check obvious alternate encodings, paths, callers, or bypasses implicated by the root cause.
- Avoid unrelated cleanup and broad refactors.
- Follow all applicable `AGENTS.md`, `SECURITY.md`, and contribution instructions.
- Modify generated code only through the repository-supported source and generation workflow.

## Verify

Derive validation commands from the target's instructions, build system, CI configuration, and changed component. Run the proportionate formatter, focused regression test, nearby tests, static analysis or linter, and build/typecheck. Record exact commands and outcomes.

Review:

- full diff from the remote default branch
- regression test failure before and success after when practical
- legitimate behavior
- nearby bypasses
- error handling and compatibility
- absence of unrelated changes or new secret logging

Do not claim verification for commands that were not run.

## Publish a ready-for-review PR

Commit intentionally, push the branch, and open a PR ready for review containing:

- a concise vulnerability class without unnecessary exploit detail
- the security invariant restored
- affected canonical and duplicate audit issues using visibility-safe references
- a summary of source and test changes
- validation commands and results
- remaining limitations or rollout considerations

End every remediation PR body with this exact Markdown line:

```markdown
_This finding was discovered by [Project Loupe](https://github.com/project-loupe/loupe)._
```

Write the multi-line PR body to a file using a safe file-editing tool. Prefer the REST pull-request endpoint with a safely created JSON request file passed to `gh api --input`; using `gh pr create --body-file "$pr_body_file"` is acceptable when repository-specific CLI behavior is required. Do not use `--draft`, `--reviewer`, or any separate initial review request. Do not pass JSON-escaped or `\\n`-encoded multi-line text through `--body`; shell quoting can preserve the backslashes and publish literal escape sequences.

Before opening the PR, inspect the body file and reject it if `rg -n -F '\n' "$pr_body_file"` finds literal backslash-`n` sequences intended as line breaks. Verify that the attribution above is the final non-empty line. After creating or editing the PR, fetch it through `GET /repos/{owner}/{repo}/pulls/{number}`. Verify that the body contains real line breaks, renders as Markdown, ends with the exact attribution, and `draft` is false. If the stored body is malformed or the attribution is absent or misplaced, repair it through the REST pull-request endpoint using a safely created request file before updating audit issues or Project state.

Do not request human reviewers when the PR is first opened. Update the Project item and all covered audit issues with the PR URL, and keep the issues open at `PR open` until gated merge and post-merge verification complete.

## Review gate

Resolve the review bot and trigger from the scope configuration. Track the PR creation time and review-bot activity during every control-loop pass. Bot activity exists when the PR has an eyes or thumbs-up reaction from the configured bot, or when that bot has posted a PR comment. If the PR has been open for at least 30 minutes with no such activity, post the configured review trigger exactly once. Record the trigger in the audit issue and do not repeat it on later passes. A trigger comment is not clean-review clearance and does not permit human review requests.

If the bot reports findings, address them in the isolated worktree, revalidate, push, and continue waiting. Treat the configured bot's `+1` reaction on the PR as the clean-review gate. Only after that reaction exists:

1. Query the PR's current GitHub reviewer suggestions through the GraphQL `suggestedReviewerActors` or `suggestedReviewers` field.
2. Exclude the PR author, bots, actors who already reviewed, and reviewers or teams already requested.
3. Intersect the remaining user suggestions with the configured automatic-review allowlist. Never automatically request a user outside the allowlist or any team. If the allowlist is empty, request nobody.
4. Request every remaining eligible suggestion in one review-request update. If GitHub suggests no eligible allowlisted user, record that and add nobody.

Do not infer suggestions from `CODEOWNERS` or commit history when the suggestion field is empty. Recheck the bot reaction, suggestions, existing reviews, and outstanding requests during later control-loop passes; never duplicate a review request.

## Merge gate

Merge a remediation PR only when the invoking prompt authorizes merging and all of these are true at the same current head SHA:

- the PR is open and ready for review
- the configured review bot has added its clean `+1` reaction
- a human reviewer has submitted `APPROVED`, and GitHub's protected-branch result recognizes that approval (`reviewDecision = APPROVED`)
- every required check is complete with an accepted conclusion
- no unresolved, non-outdated blocking review thread remains
- GitHub reports `mergeable = MERGEABLE` and `mergeStateStatus = CLEAN`

Treat GitHub's protected-branch decision as authoritative for whether the approving reviewer has the required role; do not infer sufficient authority from a username or `authorAssociation` alone. Reread the PR, reviews, threads, check rollup, and head SHA immediately before merging. If any field is pending, stale, unknown, blocked, or conflicting, do not merge.

Do not treat a non-required failed check as a merge blocker by itself. Inspect its logs, determine whether it is plausibly related to the remediation diff, and record the result. A clearly unrelated flaky or informational check may remain failed when all required checks have accepted conclusions and GitHub otherwise reports the gated PR `APPROVED`, `MERGEABLE`, and `CLEAN`; a related or unexplained failure still requires investigation before merge.

Query repository merge settings and use an allowed ordinary merge method. Never use an admin bypass, disable protections, force-push, enable auto-merge to skip a live gate, or merge a different head than the one verified. Do not manually delete branches; repository automation may apply its configured post-merge policy.

After a successful merge:

1. Fetch the current remote default branch and identify the resulting default-branch commit.
2. Verify the security fix and regression coverage are present on that exact revision; rerun the smallest meaningful post-merge check when practical.
3. Add the PR, merge commit, reviewed default-branch commit, and verification result to every covered audit issue and Project item.
4. Close every covered audit issue and set `Status = Done` only after verification succeeds.
5. Recalculate remediation WIP and start the next highest-severity eligible finding.

## WIP and follow-up

Maintain at most ten open remediation PRs unless the user changes the limit. Check PR, CI, bot reaction, suggested reviewers, approvals, unresolved threads, and merge state through REST during each control-loop pass. Use GraphQL only for a specific datum that current REST does not expose, including the configured reviewer-suggestion query. Address focused feedback in the corresponding worktree, merge promptly when the full gate is satisfied, and do not let PR follow-up block triage of unrelated findings.

When a PR cannot proceed, return the item to `Confirmed` or `Human review` with the exact blocker rather than leaving it ambiguously in `Fixing`.
