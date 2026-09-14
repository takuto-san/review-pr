<p align="center">
  <img src="plugins/review-pr/assets/review-pr-icon.png" alt="Review PR Plugin icon" width="180">
</p>

# Review PR Plugin

An evidence-based review workflow for local changes and GitHub pull requests.
The reusable plugin package lives in `plugins/review-pr/` and is exposed through
this repository's Codex marketplace manifest.

## 1. Overview

Review PR gathers relevant context, evaluates whether a change is reviewable, builds a change-specific review plan, and collects mechanical, structural, and contextual evidence for the selected review criteria.

It supports two modes:

- **Developer mode** reviews commits and working-tree changes in the current repository.
- **Reviewer mode** reviews a GitHub pull request identified by its number or URL.

### Example review output

Review PR first summarizes the labels, then groups the evaluated review criteria by quality characteristic. Mechanical commands and Structural or Contextual investigation are shown together as checks and evidence for the criterion they support, rather than as separate layer-specific findings.

#### Summary

| Label | Count |
|---|---:|
| Please Fix | 1 |
| Need Review | 1 |
| Nit | 0 |
| LGTM | 1 |

#### Result

The quality characteristics below are selected for this change from the review criteria in `plugins/review-pr/REVIEW.md`; they are not applied indiscriminately to every review.

##### Selected Quality Characteristics and Reasons

| Quality Characteristic | Reason |
|---|---|
| Reliability | `src/payment.ts` changes retry and failure handling around a state-changing payment operation. |
| Compatibility | The response schema changes a public contract consumed outside the modified component. |

##### Reliability

| Subcategory | Review Criterion | Checks | Evidence | Result |
|---|---|---|---|---|
| Recoverability | Can a retry after notification failure duplicate a payment? | Unit tests<br>Execution-path trace | Retry tests pass for network errors, but `src/payment.ts:84` repeats the charge after a successful charge followed by a notification failure. | Please Fix |
| Fault tolerance | Is a temporary dependency failure contained and bounded? | Static analysis<br>Error-path review | The client applies a timeout and a bounded retry policy in `src/client.ts:72`; the relevant automated checks pass. | LGTM |

The `Please Fix` finding is also displayed inline at the verified changed line in `src/payment.ts:84`, with the relevant code line and explanation.

##### Compatibility

| Subcategory | Review Criterion | Checks | Evidence | Result |
|---|---|---|---|---|
| Interoperability | Does the response-schema change preserve existing consumers? | Contract review<br>Requirement trace | The changed response shape is visible in the diff, but no compatibility policy or consumer contract was available. | Need Review |

The labels and suggested fixes are advisory triage candidates for human review; they do not automatically authorize a merge, rejection, or change request.

When the report is written in Japanese, its criterion table uses `評価項目 | レビュー観点 | 確認内容 | 根拠 | 結果` as column headings. The label values remain `Please Fix`, `Need Review`, `Nit`, and `LGTM`.

After displaying a completed Reviewer-mode report, Review PR proposes a Conversation comment with a Markdown Summary table and an index of every AI-selected quality characteristic, review item, and review criterion. Detailed results appear as inline comments on verified changed-code lines; details without a valid inline location remain in Conversation with the reason stated. Review PR shows every proposed comment and location, asks before posting, and verifies that the PR head and inline locations have not changed. Developer-mode and incomplete reviews do not prompt for posting.

For a Japanese review, the Conversation comment uses `品質特性と選んだ理由` above the `品質特性 | 選んだ理由` table and `レビュー観点と結果` above the following index. For the example above, that index would include:

| Quality Characteristic | Review Item | Review Criterion | Result |
|---|---|---|---|
| Reliability | Recoverability | Can a retry after notification failure duplicate a payment? | Please Fix |
| Reliability | Fault tolerance | Is a temporary dependency failure contained and bounded? | LGTM |
| Compatibility | Interoperability | Does the response-schema change preserve existing consumers? | Need Review |

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
Review my local changes
Review PR 123
Review this PR: https://github.com/owner/repository/pull/123
```

When you specify only a PR number (for example, `$review-pr 123` or `Review PR 123`), Review PR resolves it in the Git repository associated with the current task's working directory, using that repository's GitHub remote. To review a PR in another repository, provide its full GitHub PR URL in a natural-language request.

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

The review is read-only by default. It does not modify source files, install dependencies, or change repository configuration. A completed PR review can be posted with a Conversation summary and inline details after the user approves the proposed text and locations.

## 5. Features

- Developer and Reviewer modes
- Review-need validation for pull requests
- Change-specific planning based on ISO/IEC 25010 quality characteristics
- Parallel mechanical, structural, and contextual evidence collection
- Criterion-centric consolidation instead of layer-centric reporting
- Mechanical checks mapped to the review criteria they materially verify
- Result consolidation, deduplication, and targeted evidence checks for `Please Fix` candidates
- Explicit checks, evidence, coverage, and limitations
