The orchestrator executes this check directly; do not delegate an agent.

## Mission

Analyze Change Scope for the PR agent using the Google Small CLs principles. Do not review code correctness or decide whether review is needed. Determine whether the change is a minimal, self-contained unit and whether the amount and grouping of substantive change impose excessive cognitive load on a human reviewer. Do not modify files.

## Input

Use the collected review target, base branch, PR number, PR description, and related context. Obtain missing facts only through available read-only Git and GitHub CLI commands.

If the target or diff statistics cannot be established, do not guess. Record the uncertainty and avoid a stronger classification than the evidence supports.

## Analysis procedure

1. Obtain changed files, additions, deletions, and total changed lines.
2. Separate generated files, lockfiles, and simple moves or deletions from substantive review work.
3. Group changes into logical Change Groups that share one purpose.
4. Identify mixtures of features, bug fixes, refactoring, tests, configuration, or migrations.
5. Determine whether the PR addresses one purpose with the smallest practical self-contained change, rather than including an entire feature unnecessarily.
6. Check whether related tests and enough usage context are present to make the change understandable. In particular, note an API addition with no representative usage. Treat missing behavioral coverage as input to the later review plan rather than deciding code correctness here.
7. Identify safe split points. Allow both independently mergeable groups and groups that can be submitted in an explicit dependency order, provided every submitted step leaves the repository in a valid state.
8. Apply Google Small CLs principles conceptually: use line and file counts as evidence of reviewer workload, never as hard thresholds. Account for cases such as deletions and trusted mechanical refactoring that are large in raw size but cheap to review.

This validation is only about reviewer workload. It is advisory and never
blocks or skips the review. Closed, draft, trivial, and already-reviewed pull
requests are handled by `eligibility` before scope analysis runs.

## Classification

- `ok`: The PR is one minimal, self-contained, understandable change with manageable reviewer workload.
- `warning`: The PR is not sufficiently minimal, self-contained, understandable, or manageable for a fully reliable review in one pass. This is advisory and does not block review.

Do not classify a change as `warning` from raw line count alone. Account for generated content, mechanical edits, reliable bulk refactoring, conceptual complexity, and the number of execution paths a reviewer must hold in mind.

The warning status does not change review eligibility, suppress a review layer,
or map to a workflow finding label. Continue the review with available evidence
and report any reduced confidence, missing context, or unreviewed areas. When a
safe split is identifiable, describe it in `suggestion`; otherwise leave that
field empty.

## Completion criteria

- Account for every changed file in either a Change Group or an explicitly identified non-substantive category.
- Base the classification on minimality, self-containment, cohesion, understandability, and reviewer workload, not raw size alone.
- For `warning`, explain the concrete reason and, when possible, state whether the suggested split is independent or give the safe submission order.
- Continue all applicable review work regardless of the scope classification.
- Present `warning` as advisory, not as a blocker or finding.
- Do not emit code-quality findings or review-eligibility decisions.
- Record material uncertainty explicitly.

## Output

Return exactly one A2A-compatible Artifact using `name: review.scope` and
`metadata.schema: review/scope`. Put exactly the following payload in
`parts[0].data`. Do not report code problems or quality findings.

```json
{
  "status": "ok | warning",
  "stats": {
    "changed_files": 0,
    "additions": 0,
    "deletions": 0,
    "lines_changed": 0
  },
  "change_groups": [
    {
      "name": "Short concrete name",
      "purpose": "What this group changes",
      "files": 0,
      "additions": 0,
      "deletions": 0
    }
  ],
  "reason": "Required when status is warning",
  "suggestion": "Safe split and submission order when identifiable; otherwise empty",
  "uncertainties": [

  ]
}
```
