# Saved review plan contract

Codex `plan` creates one durable, private JSON bundle outside the target
repository before prompting. In Claude Code native Plan Mode, create it only
after `ExitPlanMode` approves the complete plan; recompute and compare the
presented fingerprint first. A rejected or revised native plan creates no
bundle.
Use `$XDG_STATE_HOME/review-pr/plans/<Plan ID>.json` when `XDG_STATE_HOME` is set,
otherwise `~/.local/state/review-pr/plans/<Plan ID>.json`. A Plan ID is at least
128 bits of cryptographically random lowercase hexadecimal. Create the directory
with owner-only access and write the file atomically with owner-only permissions
where supported. Never follow a symlink or replace an existing plan. If storage
or later validation fails, do not claim that the run can resume.

The bundle is an A2A-compatible Artifact:

```json
{
  "artifactId": "Unique numeric-string Artifact ID assigned after earlier stages",
  "name": "review.plan-bundle",
  "parts": [{
    "mediaType": "application/json",
    "data": {
      "plan_id": "32-or-more-random-hex-digits",
      "created_at": "ISO-8601 timestamp",
      "target_fingerprint": {
        "mode": "developer | reviewer",
        "repository": "Canonical repository identity",
        "pr_number": "Number or null",
        "base_sha": "Full SHA",
        "head_sha": "Full SHA",
        "snapshot_sha256": "64 lowercase hexadecimal digits"
      },
      "target": {},
      "eligibility": {},
      "context": {},
      "scope": {},
      "plan": {}
    }
  }],
  "metadata": {
    "targetId": "001",
    "schema": "review/plan-bundle",
    "schemaVersion": "1.0",
    "producer": "review-pr:review"
  }
}
```

The named fields `target`, `eligibility`, `context`, `scope`, and `plan` hold
the complete corresponding validated A2A Artifacts. Do not reconstruct a
missing field from conversation history. Preserve the selected criterion IDs,
source locations, expected checks, and planned human decisions. `plan_id` must
match the file name, and every nested Artifact must share `metadata.targetId`.

Build `snapshot_sha256` with SHA-256 over a deterministic, length-delimited
encoding of the review target. Include repository identity, PR number when
applicable, base and head SHAs, and the complete diff. For Developer mode also
include staged and unstaged patch bytes and every relevant untracked source
file's relative path and content bytes, sorted by path. Do not embed ignored
files or credentials in the saved bundle. Store the digest, not untracked file
contents, in `target_fingerprint`; the complete review target Artifact may
contain only the source data needed for downstream review.

At `run`, validate the envelope, schema version, Plan ID, nested Artifacts, and
criterion coverage. Recompute the fingerprint using the same procedure and
compare all identity fields exactly. Recheck PR eligibility. Reject the run
before any agent or repository command when the snapshot changed, a required
source became unavailable, or the bundle is malformed. Show the changed fields
and instruct the user to create a new plan. A plan is never silently rebased.

Approval is tied internally to the exact presented plan, although the user need
not type its Plan ID. In Claude Code, approval through `ExitPlanMode` authorizes
the native plan shown there; persist it before review. In Codex, a structured
**Approve and run** response or an explicit plain-language approval authorizes
the saved plan. Rejection ends the workflow; feedback or **Request changes**
requires a revised plan and a fresh decision.
Silence and the original request to create a plan are not approval.
Resolve the latest plan shown in the current task when
its target matches the current repository; across tasks, an explicit `run`
request may proceed only when one saved plan matches the intended repository
and target. Ask the user to choose
by target and creation time if several plans match. If the user asks to change
the plan, write a new bundle with a new ID in Codex and show its Planned Review
Coverage; in Claude Code native Plan Mode, revise before saving. An optional Plan ID can disambiguate plans. Do not mark a plan
approved inside the bundle based on inference or elapsed time.

After a completed `run`, save a result record in the same private state
directory. It references the approved Plan ID and target fingerprint and
contains consolidated criterion IDs, labels, checks, evidence locations, and
human handoffs. A later `plan` may use this record to compare review coverage
across snapshots. Keep it separate from the immutable plan bundle; never
revise a completed plan or result in place.
