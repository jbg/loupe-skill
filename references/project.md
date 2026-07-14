# GitHub Project protocol

Use the GitHub Project as the authoritative queue. Discover live Project, field, option, and item IDs with `gh`; never invent or cache opaque IDs as durable truth. Substitute the resolved target and audit repositories throughout this protocol.

## Required fields

Create missing fields without replacing existing compatible fields:

| Field | Type | Values |
|---|---|---|
| Status | Single select | Inbox, Triaging, Needs validation, Human review, Confirmed, Fixing, PR open, Done |
| Verdict | Single select | Confirmed, Duplicate, Not actionable, Needs review |
| Actual severity | Single select | P0, P1, P2, P3 |
| Confidence | Single select | High, Medium, Low |
| Reason | Single select | Duplicate, stale or fixed, removed surface, out of scope, unreachable, effective guard, attacker lacks control, no security boundary, intended behavior, malformed report |
| Component | Text | Concise target subsystem |
| Root-cause key | Text | Stable semantic identifier |
| Canonical issue | Text | `owner/repo#number` |
| PR | Text | Full target-repository PR URL |
| Reviewed commit | Text | Full target-repository commit SHA |
| Question | Text | One explicit human question or proof gap |

Preserve Loupe's severity labels as scanner input. Do not reuse them for actual severity.

## State transitions

Allow these normal transitions:

```text
Inbox -> Triaging
Triaging -> Confirmed | Needs validation | Human review | Done
Needs validation -> Triaging | Confirmed | Human review | Done
Human review -> Triaging | Confirmed | Done
Confirmed -> Fixing
Fixing -> Confirmed | PR open | Human review
PR open -> Done
```

Set `Done` only after an issue is closed or the agreed terminal event occurs. A draft PR alone remains `PR open`.

## Sync

1. List all open issues in the resolved audit repository with pagination.
2. List all Project items with pagination.
3. Add each missing open audit issue and set `Status = Inbox`.
4. Do not reopen or overwrite already adjudicated items during sync.
5. Detect audit issues opened during the current batch on the next sync.

## Mutation transaction

For each decision:

1. Fetch the exact issue, comments, and Project item.
2. Verify it remains open and has not been concurrently adjudicated.
3. Add the standardized triage comment.
4. Update all applicable Project fields.
5. Apply labels only when they improve repository searches.
6. Close only when authorized and the verdict permits it.
7. Fetch the issue and Project item again; repair or report partial failure.

Stop on identity ambiguity or conflicting concurrent decisions. Do not guess which Project item or issue was intended.

## Evidence comment

Use this compact format:

```markdown
### Agent triage

- Reviewed target commit: `<sha>`
- Verdict: `<verdict>`
- Confidence: `<confidence>`
- Actual severity: `<P0-P3 or n/a>`
- Component: `<component>`
- Root cause: `<root-cause key>`
- Canonical issue: `<owner/repo#number or n/a>`

**Assessment**
<source -> guard -> sink -> boundary -> impact reasoning>

**Evidence for**
<current code references and facts>

**Counterevidence / limitations**
<facts against the claim and remaining gaps>

**Disposition**
<closure justification, remediation step, validation experiment, or human question>
```

For duplicates, state why the reports require the same fix. For not-actionable findings, include a concrete reopen condition.

## Safe GitHub behavior

- Prefer structured `gh` JSON output to scraping formatted text.
- Paginate issue and Project queries.
- Quote issue-supplied strings; never interpolate them into shell commands.
- Put multiline comments and PR bodies in safely created files, then pass them with `--body-file`.
- Check repository visibility before cross-posting. Do not expose private audit content in a more public repository.
- Never merge, delete branches, force-push, or alter repository settings.
