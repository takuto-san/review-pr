<p align="center">
  <img src="plugins/review-pr/assets/review-pr-icon.png" alt="Review PR Plugin icon" width="180">
</p>

# Review PR Plugin

An evidence-based review workflow for local changes and GitHub pull requests.
The reusable plugin package lives in `plugins/review-pr/` and is exposed through
this repository's Codex marketplace manifest.

## 1. Overview

Review PR separates planning from review. It first gathers context, shows rubric-derived review criteria, and lists planned tests and inspections in a separate table linked by criterion ID. After you approve that plan, it collects mechanical, structural, and contextual evidence and reports which review items ran, what AI verified, what needs a fix, and what still needs human judgment.

The review has two deliberate steps:

```text
review-pr [PR number]      → inspect target, context, scope, and REVIEW.md
                           → show Planned Review Coverage
                           → request approval in the host UI
user approves the displayed plan
                           → verify the target has not changed
                           → run three-layer review
                           → show Review Criteria and Items Requiring Confirmation or Action
```

In Claude Code, Review PR uses native Plan Mode and `ExitPlanMode` to show the coverage for approval or feedback. In Codex, an available structured input prompt offers **Approve and run**, **Do not run**, and **Request changes**. Approval starts the review without another command. The Plan ID is saved internally (in Claude Code, after approval); you can also reply “Run the review using this plan”. Feedback produces a revised plan for another decision. If the preferred approval tool is unavailable, Review PR uses an available structured prompt or conversation. If several saved plans could apply, it asks which target and creation time you mean. If a PR head or local diff changed after planning, create a new plan before running.

It supports two modes:

- **Developer mode** reviews commits and working-tree changes in the current repository.
- **Reviewer mode** reviews a GitHub pull request identified by its number or URL.

### Example review output

At planning time, Review PR opens with the exact PR title and four short bullets: `Summary` explains the source-backed purpose; `Change scope` shows `Appropriate` or `Needs attention`; `Changes` gives the diff size and uses indented sub-bullets for repository-relative paths when needed; `What to check` names the behaviors or contracts the review will check. It then shows a `Review Criteria` table derived from applicable `REVIEW.md` concerns. Under `Review Items`, one table is shown per quality characteristic; each item has `No | Subcharacteristic | Review criterion | Planned check | Relevant location | Verification method`. `Relevant location` identifies a file and a specific function, branch, test case, or contract section. `Verification method` says how to check it and what result to observe. The `Review criterion` column gives users the item-to-criterion relationship without an extra numeric reference column. No repository checks or review agents run during planning, and the plan does not predict which decisions AI will defer to a human.

For example, a plan for a payment retry change could show (all paths and counts are illustrative):

### PR #123: Fix retries after post-payment notification failures

- **Summary:** Change retry handling when a notification fails after a successful payment, along with the payment API response format.
- **Change scope:** Appropriate
- **Changes:** 4 files changed (+86 lines, −24 lines).
  - `src/payment.ts`: Notification failure and retry handling after payment completion.
  - `src/payment-api.ts`: Payment result response fields.
  - `tests/payment-retry.test.ts`, `tests/payment-api.test.ts`: Update related test cases.
- **What to check:** Verify that retries after partial failures prevent duplicate payments and that responses remain compatible with existing clients.

#### Review Criteria

| No | Quality characteristic | Subcharacteristic | Review criterion | Reason |
|---|---|---|---|---|
| 1 | Reliability | Recoverability | Do retries after notification failures avoid duplicate payments? | The failure path after payment completion has changed |
| 2 | Compatibility | Interoperability | Can existing clients handle the new response format? | Public API response fields have changed |

#### Review Items

**Reliability**

| No | Subcharacteristic | Review criterion | Planned check | Relevant location | Verification method |
|---|---|---|---|---|---|
| 1 | Recoverability | Do retries after notification failures avoid duplicate payments? | Verify that retries do not call the payment API again when a notification fails after a successful payment | Notification failure and retry branches in `src/payment.ts`; related cases in `tests/payment-retry.test.ts` | Trigger a notification failure and verify that the payment API call count remains one |
| 2 | Recoverability | Do retries after notification failures avoid duplicate payments? | Verify that the successful payment state persists after a partial failure | Payment state persistence in `src/payment.ts` | Trace state transitions from payment completion through retry and check that the success state is not overwritten |

**Compatibility**

| No | Subcharacteristic | Review criterion | Planned check | Relevant location | Verification method |
|---|---|---|---|---|---|
| 3 | Interoperability | Can existing clients handle the new response format? | Check whether consumers that reference old fields fail with the new response | Response field references in `src/payment-client.ts`; payment API response contract | Compare field names before and after the change against consumer references |

For example, the final `Review Criteria` table uses the same criterion No as the plan. The review-item results are shown separately by quality characteristic. `Items Requiring Confirmation or Action` appears only after the evidence has been evaluated.

#### Summary

| Result | Count |
|---|---:|
| Please Fix | 1 |
| Need Review | 1 |
| Nit | 0 |
| LGTM | 0 |

#### Review Criteria

| No | Review criterion | Result |
|---|---|---|
| 1 | Do retries after notification failures avoid duplicate payments? | Please Fix |
| 2 | Can existing clients handle the new response format? | Need Review |

#### Review Items

**Reliability**

| No | Subcharacteristic | Review criterion | Check performed | Status | Reason |
|---|---|---|---|---|---|
| 1 | Recoverability | Do retries after notification failures avoid duplicate payments? | Retry test after a notification failure | Performed | The payment API was called twice |
| 2 | Recoverability | Do retries after notification failures avoid duplicate payments? | Payment state transition inspection | Performed | Execution enters a path that repeats a completed payment from `src/payment.ts:84` |

**Compatibility**

| No | Subcharacteristic | Review criterion | Check performed | Status | Reason |
|---|---|---|---|---|---|
| 3 | Interoperability | Can existing clients handle the new response format? | Compare response fields against consumer references | Performed | External client contracts are absent from the available documentation |

#### Items Requiring Confirmation or Action

| No | Reason | Where to check | Decision needed | What AI has verified |
|---|---|---|---|---|
| 2 | External client contracts are unknown | Payment API response specification and consumer contracts | Whether old fields must be retained | Compared repository references and responses before and after the change |

`Please Fix` items appear in a separate short list with the verified changed-code location. The detailed result tables follow.

#### Results

The quality characteristics below are selected for this change from the review criteria in `plugins/review-pr/REVIEW.md`; they are not applied indiscriminately to every review.

##### Quality Characteristics and Selection Reasons

| Quality characteristic | Selection reason |
|---|---|
| Reliability | The post-payment retry path changes in `src/payment.ts`. |
| Compatibility | Public API response fields change. |

##### Reliability

| Evaluation item | Review criterion | Checks performed | Reason | Result |
|---|---|---|---|---|
| Recoverability | Do retries after notification failures avoid duplicate payments? | Retry test<br>Execution path tracing | Execution enters a path that repeats a completed payment from `src/payment.ts:84`, and the payment API was called twice. | Please Fix |

The `Please Fix` finding is also displayed inline at the verified changed line in `src/payment.ts:84`, with the relevant code line and explanation.

##### Compatibility

| Evaluation item | Review criterion | Checks performed | Reason | Result |
|---|---|---|---|---|
| Interoperability | Can existing clients handle the new response format? | Compare response fields against consumer references | External client contracts are absent from the available documentation, so compatibility cannot be determined. | Need Review |

The labels and suggested fixes are advisory triage candidates for human review; they do not automatically authorize a merge, rejection, or change request.

When the report is written in Japanese, its criterion table uses Japanese equivalents of `Evaluation item | Review criterion | Checks performed | Reason | Result` as column headings. The label values remain `Please Fix`, `Need Review`, `Nit`, and `LGTM`.

After displaying a completed Reviewer-mode report, Review PR proposes a Conversation comment with a Markdown Summary table and an index of every AI-selected quality characteristic, review item, and review criterion. Detailed results appear as inline comments on verified changed-code lines; details without a valid inline location remain in Conversation with the reason stated. Review PR shows every proposed comment and location, asks before posting, and verifies that the PR head and inline locations have not changed. Developer-mode and incomplete reviews do not prompt for posting.

For a Japanese review, the Conversation comment uses Japanese equivalents of `Quality Characteristics and Selection Reasons` above the `Quality characteristic | Selection reason` table and `Review Criteria and Results` above the following index. For the example above, that index would include:

| Quality characteristic | Evaluation item | Review criterion | Result |
|---|---|---|---|
| Reliability | Recoverability | Do retries after notification failures avoid duplicate payments? | Please Fix |
| Compatibility | Interoperability | Can existing clients handle the new response format? | Need Review |

## 2. Architecture

### Three-layer review model

The three layers separate checks by the kind of evidence and judgment they require. They are specialized evidence providers, not three independent final reviews: their results are consolidated by review criterion into one report.

| Layer | What it examines | Typical examples |
|---|---|---|
| **Mechanical** | Facts that repository tooling can verify consistently | Formatting, lint, type checking, compilation, static analysis, and automated tests |
| **Structural** | How the changed code behaves within the wider codebase | Execution paths, architecture, dependencies, state, error handling, security, performance, and maintainability |
| **Contextual** | Whether the change matches its purpose and surrounding decisions | PR intent, requirements, acceptance criteria, compatibility policy, migrations, documentation, and user value |

For example, a passing test is Mechanical evidence. Whether the tested design handles retries safely is a Structural question. Whether retry behavior matches the stated product requirement is a Contextual question. Looking at all three prevents automated checks from being mistaken for a complete code review, while avoiding repeated work across reviewers.

This organization is inspired by Greptile's [3-Layer Code Review Checklist](https://www.greptile.com/content-library/code-review-checklist), which distinguishes Mechanical, Structural, and Narrative review. Review PR adapts the Narrative layer as **Contextual** review and implements its own criterion-centric planning, evidence collection, and consolidation workflow.

### Package structure

```text
review-pr/
├── .agents/
│   └── plugins/
│       └── marketplace.json
├── plugins/
│   └── review-pr/
│       ├── .codex-plugin/
│       │   └── plugin.json
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── assets/
│       │   ├── review-pr-icon.png
│       │   └── review-workflow.svg
│       ├── agents/
│       │   └── review/
│       │       ├── mechanical.md
│       │       ├── structural.md
│       │       └── contextual.md
│       ├── skills/
│       │   └── review-pr/
│       │       ├── SKILL.md
│       │       └── checks/
│       │           ├── artifacts.md
│       │           ├── eligibility.md
│       │           └── scope.md
│       ├── REVIEW.md
│       └── LICENSE
├── PRIVACY.md
├── README.md
├── TERMS.md
└── LICENSE
```

![Review PR workflow](plugins/review-pr/assets/review-workflow.svg)

## 3. How to Use

### Supported agents

| Agent or harness | Packaging status | How to use it |
|---|---|---|
| Codex | Native | Add this repository as a marketplace, install `review-pr`, then invoke `$review-pr` or use a matching natural-language request. |
| Claude Code | Native | Load `plugins/review-pr/`, then invoke `/review-pr` or use a matching natural-language request. |
| Other coding agents | Portable skill | Import `plugins/review-pr/skills/review-pr/SKILL.md` using the agent's skill mechanism. |

### Codex

Add this GitHub repository as a Codex plugin marketplace:

```bash
codex plugin marketplace add takuto-san/review-pr --ref main
```

Install the plugin:

```bash
codex plugin add review-pr@review-pr
```

Verify the installation:

```bash
codex plugin list
```

Then start a new Codex task in the repository you want to review and run:

```text
$review-pr
$review-pr 123
```

Natural-language requests are also supported:

```text
Plan a review of my local changes
Plan a review of PR 123
Plan a review of this PR: https://github.com/owner/repository/pull/123
This plan looks good. Run the review.  # optional reply instead of the approval choice
```

When you specify only a PR number (for example, `$review-pr 123`), Review PR resolves it in the Git repository associated with the current task's working directory, using that repository's GitHub remote. To plan a PR in another repository, provide its full GitHub PR URL in a natural-language request. A general request to review a PR starts with the planning step; it does not run review agents until you approve the plan.

To refresh the marketplace after updates:

```bash
codex plugin marketplace upgrade review-pr
```

### Claude Code

Load the plugin package directly:

```bash
claude --plugin-dir /path/to/review-pr/plugins/review-pr
```

Then run:

```text
/review-pr
/review-pr 123
```

## 4. Requirements

- A Git repository for Developer mode
- GitHub CLI (`gh`) installed and authenticated for Reviewer mode
- Access to the target repository and pull request
- Repository-defined test or analysis commands for mechanical verification

Planning and review do not modify the target's source files, install dependencies, or change repository configuration. Plans are saved privately in the user's local state directory, outside the target repository. A completed PR review can be posted with a Conversation summary and inline details after the user approves the proposed text and locations.

## 5. Features

- Developer and Reviewer modes
- Review-need validation for pull requests
- Change-specific planning based on ISO/IEC 25010 quality characteristics
- Separate plan and approved run operations, with snapshot validation before review
- Planned Review Coverage and criterion-level human handoff
- Parallel mechanical, structural, and contextual evidence collection
- Criterion-centric consolidation instead of layer-centric reporting
- Mechanical checks mapped to the review criteria they materially verify
- Result consolidation, deduplication, and targeted evidence checks for `Please Fix` candidates
- Explicit checks, evidence, coverage, and limitations
