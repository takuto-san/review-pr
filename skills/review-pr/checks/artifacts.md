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
    "producer": "review:review"
  }
}
```

Use `review.target`, `review.eligibility`, `review.context`, `review.scope`,
`review.plan`, `review.mechanical`, `review.structural`, and
`review.contextual` as artifact names for the corresponding data and stages.
The orchestrator passes required inputs in these Artifact envelopes, and each
receiver reads the typed payload from `parts[0].data`. No stage may infer
missing payload fields from conversation history.

## ID rules

The orchestrator assigns IDs and passes them explicitly to reviewers. Generated
IDs are strings containing only decimal digits, starting at `"001"`, then
`"002"`; use at least three digits (`"999"` is followed by `"1000"`). Do not
encode a type, layer, or target in an ID. These IDs are local to one review run,
not global identifiers.

| Field | Meaning | Numbering scope |
|---|---|---|
| `metadata.targetId` | The PR or local change set being reviewed | Unique within the run; map it to the repository, PR when applicable, base/head SHAs, and diff in the shared target context |
| Result `id` | One review-plan item | Unique within the target across all layers and batches |
| `metadata.batchId` | A group of at most five items delegated together | Unique within the target across structural and contextual layers |
| `artifactId` | One output Artifact | Unique across all stages and targets in the run, including consolidated outputs |

Each numbering scope starts at `"001"` independently. A repeated value in
different fields is valid. Keep assigned IDs unchanged through review and
verification; do not restart item numbering for each batch. The orchestrator
supplies each invocation's output `artifactId`, `targetId`, and applicable
`batchId`; reviewers copy them rather than generating IDs.

Requirement and acceptance-criterion IDs supplied by sources remain unchanged,
even if they contain letters or hyphens. Their source locations must also be
preserved.

## Completion requirements

- Every review-plan item has a stable `id` preserved through review and consolidation.
- Every reviewer result uses the shared A2A-compatible Artifact envelope.
- Every reviewer receives its required inputs explicitly.
- Each structural and contextual result contains exactly one result per assigned item.
- Missing evidence uses `assessment.evaluation.level: not_assessable` instead of omission.
- Every executed verification command and result is recorded.
- The orchestrator does not treat an incomplete review as complete.
