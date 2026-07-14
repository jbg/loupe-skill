# Remediation protocol

## Accept a finding for work

Require a canonical confirmed issue, actual severity, confidence, current-code evidence, expected invariant, and a reproducible test or explicit proof gap. Recheck related audit issues so one root-cause fix covers the correct cluster.

Determine the visibility of the audit and target repositories. For P0/P1, keep non-public evidence in the least exposed authorized location and obtain the required user decision before publishing exploit-enabling details to a more public target repository.

## Isolate the change

1. Resolve and refresh the target repository's default branch without disturbing the user's worktree.
2. Create a dedicated worktree from the current remote default branch.
3. Create a short security branch that follows the target's contribution rules and the host's required branch-prefix policy.
4. Assign exactly one writing agent to the worktree.
5. Preserve and avoid unrelated changes everywhere else.

Do not reuse a dirty worktree, force-reset user work, or combine unrelated findings.

## Validate before fixing

Trace the current source, control, sink, and boundary again. Prefer adapting the report's proposed test after reviewing it for safety. Confirm the test demonstrates the claimed invariant rather than merely exercising adjacent behavior.

If the finding does not reproduce, return it to triage with exact counterevidence. Do not manufacture a fix to satisfy the report.

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

## Publish a draft PR

Commit intentionally, push the branch, and open a draft PR containing:

- a concise vulnerability class without unnecessary exploit detail
- the security invariant restored
- affected canonical and duplicate audit issues using visibility-safe references
- a summary of source and test changes
- validation commands and results
- remaining limitations or rollout considerations

Update the Project item and all covered audit issues with the PR URL. Keep audit issues open at `PR open` until the agreed post-merge verification and closure step. Never merge the PR.

## WIP and follow-up

Maintain at most two open remediation PRs unless the user changes the limit. Check CI and review state during each control-loop pass. Address focused feedback in the corresponding worktree, but do not let PR follow-up block triage of unrelated findings.

When a PR cannot proceed, return the item to `Confirmed` or `Human review` with the exact blocker rather than leaving it ambiguously in `Fixing`.
