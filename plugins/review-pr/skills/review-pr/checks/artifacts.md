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
`review.plan`, `review.plan-bundle`, `review.mechanical`, `review.structural`, and
`review.contextual` as artifact names for the corresponding data and stages.
The orchestrator passes required inputs in these Artifact envelopes, and each
receiver reads the typed payload from `parts[0].data`. No stage may infer
missing payload fields from conversation history.

The durable plan bundle and target fingerprint are defined in
[plan-bundle.md](plan-bundle.md). The plan is presented before delegation; after the user approves it, the
orchestrator loads and validates the saved bundle before review. The
Plan ID is internal in normal use and optional for disambiguation.

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
- source URI and precise locator, including target-policy, plugin-baseline, or cited-requirement origin
- selection reason grounded in the target change
- primary and supporting review roles

The plan also contains review items separate from criteria. Each planned item
has a unique `review_item_id`, references one existing `criterion_id`, and
describes a concrete planned test, analysis, trace, or source inspection,
including the condition to check, a source-backed target, and a verification
method. If a precise target or method is unknown, record that gap instead of
inventing one. A criterion may have multiple items. Do not predict a
human handoff in the plan. After review, retain planned item IDs and record
whether each was performed, its actual check and evidence, or why it was not
performed. Assign a new ID to an additional check discovered during review.
Only the result may identify a remaining human decision from observed evidence.

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
IDs are strings containing only decimal digits. `criterion_id` and
`review_item_id` start at `"1"`, then `"2"`, with no leading zeros; these are
shown to users as `No`. Other internal IDs start at `"001"`, then `"002"`,
and use at least three digits (`"999"` is followed by `"1000"`). Do not
encode a type, layer, or target in an ID. These IDs are local to one review run,
not global identifiers.

| Field | Meaning | Numbering scope |
|---|---|---|
| `metadata.targetId` | The PR or local change set being reviewed | Unique within the run; map it to the repository, PR when applicable, base/head SHAs, and diff in the shared target context |
| `criterion_id` | One review-plan criterion | Unique within the target across all roles and batches |
| `review_item_id` | One planned or added review activity | Unique within the target across all criteria and roles |
| `metadata.batchId` | A group of at most five criteria delegated together | Unique within the target across structural and contextual roles |
| `artifactId` | One output Artifact | Unique across all stages and targets in the run, including consolidated outputs |

Each numbering scope starts at its specified first value independently. A repeated value in
different fields is valid. Keep assigned IDs unchanged through review and
verification; do not restart criterion numbering for each batch. The
orchestrator supplies each invocation's output `artifactId`, `targetId`, and
applicable `batchId`; reviewers copy them rather than generating IDs.

Requirement and acceptance-criterion IDs supplied by sources remain unchanged,
even if they contain letters or hyphens. Their source locations must also be
preserved.

## Completion requirements

- Every review-plan criterion has a stable `criterion_id` preserved through review and consolidation.
- Every planned review item has a stable `review_item_id` and a valid `criterion_id`; completed review items retain those IDs in the result.
- Every reviewer result uses the shared A2A-compatible Artifact envelope.
- Every reviewer receives its required inputs explicitly.
- Each structural and contextual result contains exactly one result per assigned criterion.
- Structural and contextual results keep performed `checks` separate from concrete `evidence`.
- Every mechanical criterion association references an existing review-plan criterion through `criterion_id` and records its check, assessment, and evidence.
- Missing evidence uses `assessment.evaluation.level: not_assessable` instead of omission.
- Every executed verification command and result is recorded, including commands that cannot be mapped to a review criterion.
- Final consolidation produces one user-facing result per review-plan criterion, not one result per review role.
- Planned and final coverage use the same criterion IDs; every `Need Review` item records why human judgment is needed, where to look, what to verify, and what AI already checked.
- The orchestrator does not treat an incomplete review as complete.
