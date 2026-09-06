The orchestrator executes this check directly; do not delegate an agent.

## Mission

Determine both whether the review should run and whether each review agent can
run with the available inputs and environment. This is an eligibility check,
not a code review or a Change Scope analysis. Do not modify files or execute
repository-controlled verification commands.

Apply PR review eligibility only in Reviewer mode. In Developer mode, set the
review status to `local_changes` when reviewable local changes exist. Apply
agent eligibility in both modes before delegating each agent.

## Input

For review eligibility, use the collected repository, pull request number,
state, draft status, base and head SHAs, changed files, diff statistics, and
available review metadata. Obtain missing facts only through read-only Git and
GitHub CLI commands.

For agent eligibility, use the resolved target, agent definition, available
tools and permissions, review plan when available, collected context, repository
manifests, CI configuration and status, and installed runtimes and dependencies.
Use only read-only inspection. Do not install dependencies, change configuration,
start services, or execute the project's test, build, lint, or analysis commands.

If the PR identity, state, or current head SHA cannot be established, do not guess. Return `review_required` and record the uncertainty because skipping requires positive evidence.

## Review decision procedure

Evaluate the conditions in this order and stop at the first matching condition:

1. `closed`: the pull request is closed or merged.
2. `draft`: the pull request is a draft.
3. `trivial`: the current head contains no substantive change that benefits
   from human review. Examples include an empty diff, generated artifacts only,
   lockfile-only updates, or formatting-only changes that are fully enforced by
   existing automation. Do not classify a change as trivial merely because it
   is small.
4. `already_reviewed`: the current authenticated reviewer has already submitted
   a completed review for the current head SHA and no review-relevant changes
   have been pushed since. A review of an older head, a pending review, an
   automated check, or another person's review does not satisfy this condition.
5. Otherwise, `review_required`.

Do not evaluate whether the pull request is too large, cohesive, or easy to
review. Those questions belong exclusively to `scope`.

If the evidence required to skip is missing or ambiguous, return
`review_required` and record the uncertainty. Skipping must be supported by
positive evidence.

## Agent decision procedure

Check an agent immediately before it would be delegated. A failure for one
agent does not stop agents that remain eligible.

For every agent:

1. Confirm its registered definition and required tools are available.
2. Confirm all required target data and input Artifacts are present and valid.
3. Confirm that the review plan assigns at least one applicable criterion, except
   for `mechanical`, which is driven by applicable repository checks.
4. Set `not_applicable` when no work is assigned. This is not an incomplete
   review.
5. Set `unavailable` when applicable work exists but a required input, tool,
   permission, runtime, dependency, configuration value, or service is missing.
   Record the affected checks or review-plan criterion IDs so they can be reported as
   `Unable to Verify`.
6. Set `ready` only when the agent can start with all applicable work. Use
   `partial` only for `mechanical` when at least one applicable command can run
   and at least one cannot; start it with the runnable commands and record the
   others as unavailable.

For `mechanical`, discover applicable commands from repository guidance,
manifests, build files, and CI configuration. For each command, confirm the
required executable, installed dependencies, configuration, permissions, and
external services without executing the command. Existing CI results may supply
evidence when local execution is unavailable. Pass only commands classified as
runnable to the mechanical agent.

For `structural`, require a valid target, base and head SHAs, changed files,
complete diff, and at least one assigned structural criterion. For `contextual`,
require a valid target, changed files, complete diff, collected context, and at
least one assigned contextual criterion. Missing specification evidence for an
otherwise runnable contextual criterion is handled by that reviewer as
`not_assessable`; it does not by itself make the agent unavailable.

## Completion criteria

- Evaluate conditions in the documented order and stop at the first match.
- Set `should_review` consistently with `review_status`.
- Support every skip decision with positive evidence.
- Report uncertainties without turning them into a skip decision.
- Check every applicable agent before delegation and set `should_run`
  consistently with its status.
- Do not delegate agents whose status is `unavailable` or `not_applicable`.
- Preserve unavailable checks and assigned criterion IDs for the final report.

## Output

Return exactly one A2A-compatible Artifact using `name: review.eligibility` and
`metadata.schema: review/eligibility`. Put exactly the following payload in
`parts[0].data`. Do not report code problems or quality findings.

```json
{
  "review_status": "review_required | local_changes | closed | draft | trivial | already_reviewed",
  "should_review": "true | false",
  "reason": "Short evidence-based explanation",
  "evidence": {
    "pr_state": "OPEN | CLOSED | MERGED",
    "is_draft": false,
    "head_sha": "Full head SHA",
    "substantive_changes": "true | false | unknown",
    "current_reviewer": "Login or unknown",
    "reviewed_head_sha": "Full SHA or none"
  },
  "agents": {
    "mechanical": {
      "status": "ready | partial | unavailable | not_applicable",
      "should_run": "true | false",
      "reason": "Evidence-based execution decision",
      "runnable_checks": [
        {
          "name": "unit-tests",
          "command": "Repository-defined command",
          "evidence": "Why the command can run"
        }
      ],
      "unavailable_checks": [
        {
          "name": "integration-tests",
          "reason": "Missing prerequisite"
        }
      ]
    },
    "structural": {
      "status": "ready | unavailable | not_applicable",
      "should_run": "true | false",
      "criterion_ids": ["001"],
      "missing_prerequisites": []
    },
    "contextual": {
      "status": "ready | unavailable | not_applicable",
      "should_run": "true | false",
      "criterion_ids": ["002"],
      "missing_prerequisites": []
    }
  },
  "uncertainties": []
}
```

Agent entries may be added as their inputs become available; complete the
Artifact before result consolidation. Set `should_review` to `true` for
`review_required` and `local_changes`, and to `false` for every other status.
Set an agent's `should_run` to `true` only for `ready`, or for `mechanical` when
its status is `partial`.
