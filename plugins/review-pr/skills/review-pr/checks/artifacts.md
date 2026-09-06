# Review artifact contract

Every delegated reviewer returns one A2A-compatible `Artifact` JSON object.
Unless a reviewer-specific example includes `artifactId`, it shows only the
payload placed in `parts[0].data`.

```json
{
  "artifactId": "001",
  "name": "review.context",
  "parts": [
    {
      "mediaType": "application/json",
      "data": {}
    }
  ],
  "metadata": {
    "targetId": "001",
    "schema": "review/context",
    "schemaVersion": "1.0",
    "producer": "review-pr:review"
  }
}
```

Use `review.target`, `review.eligibility`, `review.context`, `review.scope`,
`review.plan`, `review.mechanical`, `review.structural`, and
`review.contextual` as artifact names for the corresponding data and stages.
The orchestrator passes required inputs in these Artifact envelopes, and each
receiver reads the typed payload from `parts[0].data`. No stage may infer
missing payload fields from conversation history.

## Criterion-centric review model

The review plan is the source of truth for the final report. Every selected
review-plan item represents one concrete review criterion for the current
change. Mechanical, structural, and contextual work provide checks and evidence
for those criteria; they are evidence sources, not separate final-report rows.

Each review-plan item must preserve:

- `criterion_id` — stable review-criterion identifier
- `rubric.category` — quality characteristic
- `rubric.subcategory` — quality subcharacteristic
- `rubric.criterion` — source criterion selected from `REVIEW.md`
- `rubric.question` — concrete PR-specific review criterion/question
- primary and supporting review roles
- expected checks and evidence when known

Reviewer outputs must keep `checks` separate from `evidence`:

- `checks` describe what verification was actually performed, such as `Unit tests`, `Static analysis`, `Execution path trace`, or `Requirement trace`.
- `evidence` records the concrete observation produced by those checks, including command output, source locations, or code locations.

Do not use `mechanical`, `structural`, or `contextual` as user-facing check names.
Those values may remain in Artifact metadata for orchestration and traceability.

Mechanical results may support, contradict, or be inconclusive for one or more
review-plan criteria. Every such association must reference the stable
`criterion_id`. The association's `assessment` records whether the mechanical
observation `supports`, `contradicts`, or is `inconclusive` for that criterion.
A passing command must not be treated as evidence for an unrelated criterion
merely because both concern the same file or quality category.

## ID rules

The orchestrator assigns IDs and passes them explicitly to reviewers. Generated
IDs are strings containing only decimal digits, starting at `"001"`, then
`"002"`; use at least three digits (`"999"` is followed by `"1000"`). Do not
encode a type, layer, or target in an ID. These IDs are local to one review run,
not global identifiers.

| Field | Meaning | Numbering scope |
|---|---|---|
| `metadata.targetId` | The PR or local change set being reviewed | Unique within the run; map it to the repository, PR when applicable, base/head SHAs, and diff in the shared target context |
| `criterion_id` | One review-plan criterion | Unique within the target across all roles and batches |
| `metadata.batchId` | A group of at most five criteria delegated together | Unique within the target across structural and contextual roles |
| `artifactId` | One output Artifact | Unique across all stages and targets in the run, including consolidated outputs |

Each numbering scope starts at `"001"` independently. A repeated value in
different fields is valid. Keep assigned IDs unchanged through review and
verification; do not restart criterion numbering for each batch. The
orchestrator supplies each invocation's output `artifactId`, `targetId`, and
applicable `batchId`; reviewers copy them rather than generating IDs.

Requirement and acceptance-criterion IDs supplied by sources remain unchanged,
even if they contain letters or hyphens. Their source locations must also be
preserved.

## Completion requirements

- Every review-plan criterion has a stable `criterion_id` preserved through review and consolidation.
- Every reviewer result uses the shared A2A-compatible Artifact envelope.
- Every reviewer receives its required inputs explicitly.
- Each structural and contextual result contains exactly one result per assigned criterion.
- Structural and contextual results keep performed `checks` separate from concrete `evidence`.
- Every mechanical criterion association references an existing review-plan criterion through `criterion_id` and records its check, assessment, and evidence.
- Missing evidence uses `assessment.evaluation.level: not_assessable` instead of omission.
- Every executed verification command and result is recorded, including commands that cannot be mapped to a review criterion.
- Final consolidation produces one user-facing result per review-plan criterion, not one result per review role.
- The orchestrator does not treat an incomplete review as complete.
