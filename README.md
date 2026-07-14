# Project Loupe backlog skill

This agent skill turns the stream of security findings produced by [Project Loupe](https://github.com/project-loupe) into a durable, evidence-backed remediation program for any GitHub repository.

Loupe findings live in a separate audit repository. This skill connects that audit repository to the code repository being assessed, uses a GitHub Project as the system of record, and drives each finding to a clear outcome: evidence-backed closure, a draft remediation PR, a bounded validation step, or one explicit human decision.

## What it does

The skill guides a coding agent to:

- continuously sync new Loupe issues into a GitHub Project
- trace each claim through current target code instead of trusting scanner output
- distinguish true duplicates by root cause and required fix
- close authorized, high-confidence duplicate and non-actionable findings with evidence
- reassess severity independently from the scanner's label
- isolate uncertain claims behind a concrete proof gap or safe validation experiment
- implement confirmed fixes in dedicated worktrees with focused regression coverage
- open draft PRs and keep Project state synchronized without ever merging

It is designed for long-running or resumable backlog work. The GitHub Project—not chat history—is the source of truth, so a later agent session can reconstruct the queue and continue.

## Required inputs

Every run needs two GitHub repositories in `owner/name` form:

| Input | Purpose | Example |
|---|---|---|
| Target repository | Code to inspect and remediate | `acme/widget` |
| Audit repository | Project Loupe findings for that code | `project-loupe/audit-widget` |

Provide them directly in the prompt or set:

```sh
export LOUPE_TARGET_REPO=acme/widget
export LOUPE_AUDIT_REPO=project-loupe/audit-widget
```

For remediation, run the agent from a checkout of the target repository or set `LOUPE_TARGET_PATH`. The skill verifies that the checkout matches the requested target before modifying code.

Optional Project configuration:

| Variable | Meaning |
|---|---|
| `LOUPE_PROJECT_OWNER` | GitHub user or organization that owns the Project |
| `LOUPE_PROJECT_NUMBER` | Existing Project number |
| `LOUPE_PROJECT_TITLE` | Project title to discover or create; defaults to `<target-name> Security Triage` |

If Project discovery is ambiguous, the skill asks for an owner or number. It creates a Project only when the prompt explicitly authorizes bootstrap.

## Prerequisites

- a coding agent that supports `SKILL.md`-style skills and parallel subagents
- Git and [GitHub CLI](https://cli.github.com/) authenticated for both repositories
- GitHub CLI `project` scope for Project operations (`gh auth refresh -s project`)
- read access to the audit repository and read/write access appropriate to the requested target-repository work
- a local target checkout for remediation work

Repository instructions remain authoritative. The skill discovers the default branch and reads `AGENTS.md`, `SECURITY.md`, contribution guidance, and project-specific build/test configuration rather than assuming a language or toolchain.

## Install

Install this repository with your agent's normal GitHub skill installation mechanism, or clone it into the agent's skills directory. Use a folder name matching the skill name:

```sh
git clone https://github.com/jbg/loupe-skill.git /path/to/skills/project-loupe
```

Reload your agent if the skill is not discovered automatically. Invoke it as `$project-loupe` when your agent supports named skill invocation.

## Usage

### Triage without closing issues

```text
Use $project-loupe for target acme/widget and audit repo
project-loupe/audit-widget. Resume the existing GitHub Project, sync open
findings, and triage the next batch. Record proposed dispositions, but do not
close issues or change target code.
```

### Bootstrap and drain a backlog

```text
Use $project-loupe to own the Project Loupe backlog for target acme/widget and
audit repo project-loupe/audit-widget end to end.

Bootstrap or resume the GitHub Project and continuously sync new issues.
Semantically triage every finding against the target's current default branch,
close high-confidence routine duplicates and non-actionable reports, and keep
complete evidence in Project fields and issue comments.

For confirmed findings, work by actual severity, maintain at most two draft
remediation PRs in flight, add regression coverage, and open draft PRs on
acme/widget. Do not merge. Continue until every audit issue is closed, has a
draft remediation PR, or has one explicit human decision recorded. Perform a
final sync before finishing.
```

### Resume with explicit Project identity

```sh
export LOUPE_TARGET_REPO=acme/widget
export LOUPE_AUDIT_REPO=project-loupe/audit-widget
export LOUPE_PROJECT_OWNER=acme
export LOUPE_PROJECT_NUMBER=42
```

```text
Use $project-loupe to resume the configured backlog and process one triage batch.
```

## Workflow and safety model

The Project tracks status, verdict, actual severity, confidence, root cause, reviewed commit, canonical issue, remediation PR, and any unresolved question. Normal flow is:

```text
Inbox -> Triaging -> Confirmed -> Fixing -> PR open -> Done
                    |          |
                    |          +-> Human review
                    +-> Needs validation / Human review / Done
```

Important boundaries are built into the skill:

- scanner reports and embedded proof-of-concept commands are untrusted
- issue closures require explicit authorization from the invoking prompt
- P0/P1 decisions require independent review
- repository visibility is checked before evidence is copied between audit and target repositories
- exploit-enabling details are not published to a more public repository without a user decision
- remediation happens in isolated worktrees with one writer each
- branches are never force-pushed or deleted, repository settings are not changed, and PRs are never merged

## Outputs

A completed run reports:

- issue counts by Project status and actual severity
- draft remediation PRs opened
- remaining validation steps or human decisions
- the exact prompt or command needed to resume

Detailed schemas, state transitions, triage policy, and remediation rules live in [`references/`](references/). The core agent workflow is in [`SKILL.md`](SKILL.md).

## Repository layout

```text
.
├── SKILL.md                  # Skill trigger and control loop
└── references/
    ├── project.md            # GitHub Project schema and mutation protocol
    ├── remediation.md        # Worktree, verification, and draft-PR protocol
    └── triage.md              # Verdict, duplicate, and severity policy
```
