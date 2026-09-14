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
                           → show レビュー観点 and 確認・対応が必要な事項
```

In Claude Code, Review PR uses native Plan Mode and `ExitPlanMode` to show the coverage for approval or feedback. In Codex, an available structured input prompt offers **Approve and run**, **Do not run**, and **Request changes**. Approval starts the review without another command. The Plan ID is saved internally (in Claude Code, after approval); you can also reply “このプランでレビューして”. Feedback produces a revised plan for another decision. If the preferred approval tool is unavailable, Review PR uses an available structured prompt or conversation. If several saved plans could apply, it asks which target and creation time you mean. If a PR head or local diff changed after planning, create a new plan before running.

It supports two modes:

- **Developer mode** reviews commits and working-tree changes in the current repository.
- **Reviewer mode** reviews a GitHub pull request identified by its number or URL.

### Example review output

At planning time, Review PR opens with the exact PR title and four short bullets: `概要` explains the source-backed purpose; `変更範囲` shows `適切` or `要注意`; `変更内容` gives the diff size and uses indented sub-bullets for repository-relative paths when needed; `確認すること` names the behaviors or contracts the review will check. It then shows a `レビュー観点` table derived from applicable `REVIEW.md` concerns. Under `レビュー項目`, one table is shown per quality characteristic; each item has `No | 副特性 | レビュー観点 | 確認内容 | 該当箇所 | 確認方法`. `該当箇所` identifies a file and a specific function, branch, test case, or contract section. `確認方法` says how to check it and what result to observe. The `レビュー観点` column gives users the item-to-criterion relationship without an extra numeric reference column. No repository checks or review agents run during planning, and the plan does not predict which decisions AI will defer to a human.

For example, a plan for a payment retry change could show (all paths and counts are illustrative):

### PR #123：決済後の通知失敗時の再試行を修正

- **概要：** 決済成功後に通知が失敗したときの再試行処理と、決済APIの応答形式を変更する。
- **変更範囲：** 適切
- **変更内容：** 4ファイルを変更（+86行、−24行）。
  - `src/payment.ts`：決済確定後の通知失敗・再試行処理。
  - `src/payment-api.ts`：決済結果の応答フィールド。
  - `tests/payment-retry.test.ts`、`tests/payment-api.test.ts`：関連ケースを更新。
- **確認すること：** 部分失敗後の二重決済防止と、既存クライアントとの応答互換性を確認する。

#### レビュー観点

| No | 品質特性 | 副特性 | レビュー観点 | 理由 |
|---|---|---|---|---|
| 1 | 信頼性 | 回復性 | 通知失敗後の再試行で二重決済しないか | 決済確定後の失敗経路が変更された |
| 2 | 互換性 | 相互運用性 | 新しい応答形式を既存クライアントが扱えるか | 公開APIの応答フィールドが変更された |

#### レビュー項目

**信頼性**

| No | 副特性 | レビュー観点 | 確認内容 | 該当箇所 | 確認方法 |
|---|---|---|---|---|---|
| 1 | 回復性 | 通知失敗後の再試行で二重決済しないか | 決済成功後に通知が失敗しても、再試行で決済APIが再呼び出しされないか | `src/payment.ts`の通知失敗・再試行分岐、`tests/payment-retry.test.ts`の該当ケース | 通知失敗を発生させ、決済APIの呼び出しが1回のままか確認する |
| 2 | 回復性 | 通知失敗後の再試行で二重決済しないか | 部分失敗後も決済成功状態が保持されるか | `src/payment.ts`の決済状態保存処理 | 決済確定から再試行までの状態遷移を追い、成功状態が上書きされないか確認する |

**互換性**

| No | 副特性 | レビュー観点 | 確認内容 | 該当箇所 | 確認方法 |
|---|---|---|---|---|---|
| 3 | 相互運用性 | 新しい応答形式を既存クライアントが扱えるか | 旧フィールドを参照する利用側が新しい応答で失敗しないか | `src/payment-client.ts`の応答参照、決済APIの応答契約 | 変更前後のフィールド名と利用側の参照箇所を照合する |

For example, the final `レビュー観点` table uses the same criterion No as the plan. The review-item results are shown separately by quality characteristic. `確認・対応が必要な事項` appears only after the evidence has been evaluated.

#### 概要

| 結果 | 件数 |
|---|---:|
| Please Fix | 1 |
| Need Review | 1 |
| Nit | 0 |
| LGTM | 0 |

#### レビュー観点

| No | レビュー観点 | 結果 |
|---|---|---|
| 1 | 通知失敗後の再試行で二重決済しないか | Please Fix |
| 2 | 新しい応答形式を既存クライアントが扱えるか | Need Review |

#### レビュー項目

**信頼性**

| No | 副特性 | レビュー観点 | 実施内容 | ステータス | 理由 |
|---|---|---|---|---|---|
| 1 | 回復性 | 通知失敗後の再試行で二重決済しないか | 通知失敗時の再試行テスト | 実施 | 決済APIが2回呼ばれた |
| 2 | 回復性 | 通知失敗後の再試行で二重決済しないか | 決済状態の遷移確認 | 実施 | `src/payment.ts:84`から確定済み決済の再実行経路に入る |

**互換性**

| No | 副特性 | レビュー観点 | 実施内容 | ステータス | 理由 |
|---|---|---|---|---|---|
| 3 | 相互運用性 | 新しい応答形式を既存クライアントが扱えるか | 応答フィールドと利用箇所の照合 | 実施 | 外部クライアントの契約が資料にない |

#### 確認・対応が必要な事項

| No | 理由 | 確認箇所 | 判断すること | AIが確認済みのこと |
|---|---|---|---|---|
| 2 | 外部クライアントの契約が不明 | 決済APIの応答仕様と利用側の契約 | 旧フィールドを維持する必要があるか | リポジトリ内の参照箇所と変更前後の応答を照合 |

`Please Fix` items appear in a separate short list with the verified changed-code location. The detailed result tables follow.

#### 結果

The quality characteristics below are selected for this change from the review criteria in `plugins/review-pr/REVIEW.md`; they are not applied indiscriminately to every review.

##### 品質特性と選んだ理由

| 品質特性 | 選んだ理由 |
|---|---|
| 信頼性 | `src/payment.ts`で決済後の再試行経路が変わるため。 |
| 互換性 | 公開APIの応答フィールドが変わるため。 |

##### 信頼性

| 評価項目 | レビュー観点 | 確認内容 | 理由 | 結果 |
|---|---|---|---|---|
| 回復性 | 通知失敗後の再試行で二重決済しないか | 再試行テスト<br>実行経路の追跡 | `src/payment.ts:84`から確定済み決済の再実行経路に入り、決済APIが2回呼ばれた。 | Please Fix |

The `Please Fix` finding is also displayed inline at the verified changed line in `src/payment.ts:84`, with the relevant code line and explanation.

##### 互換性

| 評価項目 | レビュー観点 | 確認内容 | 理由 | 結果 |
|---|---|---|---|---|
| 相互運用性 | 新しい応答形式を既存クライアントが扱えるか | 応答フィールドと利用箇所の照合 | 外部クライアントの契約が資料になく、互換性を判定できない。 | Need Review |

The labels and suggested fixes are advisory triage candidates for human review; they do not automatically authorize a merge, rejection, or change request.

When the report is written in Japanese, its criterion table uses `評価項目 | レビュー観点 | 確認内容 | 理由 | 結果` as column headings. The label values remain `Please Fix`, `Need Review`, `Nit`, and `LGTM`.

After displaying a completed Reviewer-mode report, Review PR proposes a Conversation comment with a Markdown Summary table and an index of every AI-selected quality characteristic, review item, and review criterion. Detailed results appear as inline comments on verified changed-code lines; details without a valid inline location remain in Conversation with the reason stated. Review PR shows every proposed comment and location, asks before posting, and verifies that the PR head and inline locations have not changed. Developer-mode and incomplete reviews do not prompt for posting.

For a Japanese review, the Conversation comment uses `品質特性と選んだ理由` above the `品質特性 | 選んだ理由` table and `レビュー観点と結果` above the following index. For the example above, that index would include:

| 品質特性 | 評価項目 | レビュー観点 | 結果 |
|---|---|---|---|
| 信頼性 | 回復性 | 通知失敗後の再試行で二重決済しないか | Please Fix |
| 互換性 | 相互運用性 | 新しい応答形式を既存クライアントが扱えるか | Need Review |

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
