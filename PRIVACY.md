# Privacy Policy

Last updated: 2026-09-14

Review PR is a local code-review skill. It does not operate a hosted service,
collect telemetry, or transmit user data on its own.

When a review plan is created, the skill saves a private local plan bundle in
`$XDG_STATE_HOME/review-pr/plans/` or `~/.local/state/review-pr/plans/`. It
contains the target's review metadata, relevant diff and context, selected
review criteria, and a fingerprint used to detect changes before review. A
completed review also saves its criterion-level results there for later
comparison. These files stay on the user's device unless the user or a host
tool shares them. Review PR does not add them to the reviewed repository.

The skill may use tools already available in the host environment, such as Git
and the GitHub CLI, only to perform the review requested by the user. Any data
handling, authentication, network access, and retention performed by those
tools are governed by the host environment and the applicable third-party
services.

Questions about this policy can be raised through the repository's issue
tracker.
