# Semantic triage protocol

## Contents

1. Required record
2. Verdicts
3. Duplicate test
4. Not-actionable test
5. Severity
6. Skeptical review

## Required record

Return one record per issue:

```yaml
issue: <audit-owner>/<audit-repo>#123
target_repo: <target-owner>/<target-repo>
reviewed_commit: full-sha
claim: concise vulnerability claim
attacker_controlled_source: input and controlling actor
security_boundary: supported boundary allegedly crossed
existing_control: relevant guard or absence
sink: dangerous operation or exposed asset
reachability: current path from source to sink
impact: concrete confidentiality, integrity, or availability effect
preconditions: essential configuration, privileges, and interaction
evidence_for:
  - current code reference and fact
evidence_against:
  - current code reference and fact
verdict: Confirmed | Duplicate | Not actionable | Needs review
confidence: High | Medium | Low
actual_severity: P0 | P1 | P2 | P3 | null
component: subsystem
root_cause_key: component/source/control/sink/invariant
duplicate_candidates:
  - issue identifier
canonical_issue: issue identifier or null
reason: disposition reason or null
proof_gap: exact unresolved question or null
recommended_action: closure, validation, policy decision, or remediation
```

Use full current-code paths and line numbers in prose evidence when practical.

## Verdicts

- `Confirmed`: current evidence establishes attacker control, reachability, a supported boundary crossing, and meaningful impact under stated preconditions.
- `Duplicate`: another issue represents the same actual vulnerability and should be the canonical remediation record.
- `Not actionable`: current evidence rules out the claim or places it outside the target's supported security model.
- `Needs review`: evidence is insufficient because a runtime, deployment, or product-policy fact is genuinely unresolved.

Keep finding validity separate from remediation readiness. A finding may be `Confirmed` while its Project `Status` is `Human review` because the smallest adequate fix is a substantial feature or requires a product, UX, policy, compatibility, or architecture decision. Do not downgrade or close such a finding merely because the implementation is large. Record the exact decision needed in `proof_gap` or `recommended_action` and in the Project question field.

Do not use low confidence to downgrade a finding. Use `Needs review` with a proof gap.

## Duplicate test

Treat two reports as duplicates only when they materially share:

1. attacker-controlled source or equivalent entry point
2. missing, broken, or bypassed control
3. reachable sink
4. violated security invariant
5. impact and essential preconditions
6. one coherent corrective change

Do not infer duplication from the same file, CWE, title, scanner fingerprint, impact category, or suggested mitigation alone. Different manifestations may form one duplicate cluster when a centralized fix resolves all of them. Similar-looking reports remain separate when they need materially different fixes or cross different boundaries.

Choose the canonical issue with the freshest applicable revision, clearest root cause, strongest reproducer, and most complete impact statement. Do not automatically choose the oldest or newest.

## Not-actionable test

Use a specific reason:

- stale or fixed: current code no longer contains the vulnerable behavior
- removed surface: the shipped feature and equivalent replacement path are absent
- out of scope: the claim relies on a boundary explicitly excluded by the target's security policy
- unreachable: attacker-controlled input cannot reach the sink under supported operation
- effective guard: an existing control blocks the claimed path
- attacker lacks control: the prerequisite requires authority equivalent to or greater than the impact
- no security boundary: behavior is intended within the same trust domain
- intended behavior: the feature explicitly promises the reported capability and does not bypass a separate control
- malformed report: no coherent, testable security claim can be recovered

Changed, moved, generated, test-only, experimental, or opt-in code is context rather than an automatic reason. Explain why that context defeats or preserves the claim.

Every not-actionable result needs current-code evidence and a reopen condition.

## Severity

Assess impact, exploitability, exposure, required interaction, privileges, blast radius, and existing mitigations. Interpret these examples within the target's documented threat model; do not use CWE as severity.

| Level | General interpretation |
|---|---|
| P0 | Default-reachable unauthenticated code execution, mass credential compromise, or similarly catastrophic impact with little or no interaction |
| P1 | Credible command execution, credential disclosure, authentication or approval bypass, or sandbox escape under normal use |
| P2 | Meaningful but conditional boundary violation, scoped disclosure or integrity impact, or serious denial of service requiring interaction or opt-in configuration |
| P3 | Limited or local denial of service, minor disclosure, defense-in-depth weakness, or impact requiring privileges comparable to the result |

Rank confirmed findings by realistic exploitability within severity. Keep `Needs review` in a separate queue. If the target publishes its own severity rubric, map evidence to that rubric and record the mapping rather than silently changing these Project values.

## Skeptical review

Give the skeptical agent the raw issue and current repository, not the first agent's desired conclusion. Ask it to find:

- missing attacker control
- unreachable paths
- guards or sanitization omitted by the report
- incompatible deployment assumptions
- weaker real impact
- distinct fixes that disprove duplication
- alternate paths that preserve a supposedly fixed or removed issue

The adjudicator compares evidence, resolves disagreements, and records uncertainty. Agent agreement without independent evidence is not validation.
