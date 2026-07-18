# GitHub Project protocol

Use the GitHub Project as the authoritative queue. Discover live Project, field, option, and item IDs through the versioned REST API; never invent or cache opaque IDs as durable truth. Substitute the resolved target and audit repositories throughout this protocol.

## REST transport

Send routine GitHub operations through `gh api` with this header:

```text
X-GitHub-Api-Version: 2026-03-10
```

Do not use `gh project`, `gh api graphql`, or GraphQL-backed convenience reads in the repeated control loop when REST covers the operation. The REST `core` quota is separate from the GraphQL quota.

Resolve whether the Project owner is an organization or user, then use the matching route family:

| Operation | Organization-owned Project | User-owned Project |
|---|---|---|
| List Projects | `GET /orgs/{owner}/projectsV2` | `GET /users/{owner}/projectsV2` |
| Get Project | `GET /orgs/{owner}/projectsV2/{number}` | `GET /users/{owner}/projectsV2/{number}` |
| List fields | `GET /orgs/{owner}/projectsV2/{number}/fields` | `GET /users/{owner}/projectsV2/{number}/fields` |
| List items | `GET /orgs/{owner}/projectsV2/{number}/items` | `GET /users/{owner}/projectsV2/{number}/items` |
| Get item | `GET /orgs/{owner}/projectsV2/{number}/items/{item_id}` | `GET /users/{owner}/projectsV2/{number}/items/{item_id}` |
| Add issue or PR | `POST /orgs/{owner}/projectsV2/{number}/items` | `POST /users/{owner}/projectsV2/{number}/items` |
| Update item fields | `PATCH /orgs/{owner}/projectsV2/{number}/items/{item_id}` | `PATCH /users/{owner}/projectsV2/{number}/items/{item_id}` |

Use standard repository REST endpoints for repository metadata, issues, comments, labels, pull requests, reviews, reactions, commits, and checks. Use local Git data instead of GitHub API calls when it is already current and authoritative for code inspection.

### Efficient reads

- Request `per_page=100` and follow `Link` headers until exhausted.
- On Project item reads, use `q` to filter the queue and `fields` to request only the fields needed for the decision.
- List the Project schema once per run. Build an ephemeral name-to-ID and option-name-to-ID map; discard it when the run ends.
- Fetch the selected issue bodies and comments once in the coordinator, then pass bounded snapshots to triage subagents.
- Retain the `ETag` for repeated full-list, schema, and exact-item reads. Send `If-None-Match` on later reads; reuse the prior response only after an authenticated `304 Not Modified`.
- Use `GET /rate_limit` for quota status. If GraphQL is exhausted, continue REST-backed work rather than waiting.

### Writes

- Add an issue or PR using the REST Project-items endpoint and its repository/name/number or content-ID form.
- Update all fields for one transaction in one `PATCH`. The body contains a `fields` array; each entry has the numeric Project field `id` and a `value`. Use the option ID for a single-select field, the direct value for text/number/date, and `null` to clear a field.
- After a write, fetch that exact item through REST with the updated field IDs in `fields` and verify every intended field. Do not relist the full Project after each mutation.
- Serialize content-generating mutations and pause between large runs of writes to avoid secondary limits.

### GraphQL exception

Use GraphQL only when current official API version `2026-03-10` has no REST equivalent, such as initial Project creation or creation of a custom Project field not exposed by REST. Before using it:

1. Confirm the missing REST capability rather than assuming a high-level `gh` command requires GraphQL.
2. Check the GraphQL bucket through `GET /rate_limit`.
3. Query only the required object and fields with the smallest connection limits possible.
4. Keep the result in the run-scoped cache and do not repeat the query in subagents.
5. Record the operation and reason in the run summary when it consumes more than a trivial number of points.

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
Confirmed -> Fixing | Human review
Fixing -> Confirmed | PR open | Human review
PR open -> Done
```

Set `Done` only after an issue is closed or the agreed terminal event occurs. An open remediation PR remains `PR open`.

## Sync

1. List all open issues in the resolved audit repository through REST with pagination.
2. List all Project items through REST with pagination, requesting only the identity and queue fields needed for reconciliation.
3. Add each missing open audit issue and set `Status = Inbox`.
4. Do not reopen or overwrite already adjudicated items during sync.
5. Detect audit issues opened during the current batch on the next sync.

## Mutation transaction

For each decision:

1. Fetch the exact issue, comments, and Project item.
2. Verify it remains open and has not been concurrently adjudicated.
3. Add the standardized triage comment.
4. Update all applicable Project fields in one REST `PATCH`.
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

For duplicates, explicitly state why the reports require the same fix. For not-actionable findings, include a concrete reopen condition.

## Safe GitHub behavior

- Prefer structured REST JSON output from `gh api` to scraping formatted text.
- Always select REST API version `2026-03-10` for Projects v2 endpoints.
- Do not use GraphQL for issue or Project reads that REST supports.
- Paginate issue and Project queries.
- Cache IDs and ETags only for the current run; never treat the cache as durable state.
- Quote issue-supplied strings; never interpolate them into shell commands.
- Put multi-line comments and PR bodies in files created with safe file-editing tools, then pass them with `--body-file` or embed them in a safely created JSON request file supplied through `gh api --input`.
- End every remediation PR body with the exact italic Project Loupe attribution specified in `remediation.md`, and verify it after publication.
- Open remediation PRs ready for review without initial reviewer requests. Verify `isDraft` is false after creation or body edits.
- Request GitHub-suggested reviewers only after the configured review bot has added a `+1` reaction indicating no further findings, and only when those reviewers appear in the configured automatic-review allowlist.
- Merge only after a qualifying human approval makes GitHub report the current head approved and cleanly mergeable, required checks pass, and no blocking review thread remains. Never use an admin bypass.
- Check repository visibility before cross-posting. Do not expose private audit content in a more public target repository beyond what remediation requires.
- Never manually delete branches, force-push, weaken branch protection, enable a bypass, or alter repository settings.
