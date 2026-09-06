# Review PR Plugin

<!-- English | [日本語](docs/README.ja.md) | [简体中文](docs/README.zh-CN.md) -->

An evidence-based review workflow for local changes and GitHub pull requests.
The reusable workflow lives in `skills/review-pr/`; native plugin manifests make
it available to supported coding agents without making the review behavior
depend on any single agent.

## Table of Contents

- [1. Overview](#1-overview)
- [2. Agent Compatibility](#2-agent-compatibility)
- [3. Architecture](#3-architecture)
  - [Directory Structure](#directory-structure)
  - [Review Workflow](#review-workflow)
- [4. How to Use](#4-how-to-use)
  - [Codex](#codex)
  - [Claude Code](#claude-code)
  - [Requirements](#requirements)
  - [Features](#features)

## 1. Overview

The Review PR Plugin gathers relevant context, evaluates whether a change is reviewable, builds a change-specific review plan, and collects mechanical, structural, and contextual evidence for the selected review criteria. The orchestrator consolidates that evidence by review criterion and checks findings that may require a fix.

It supports two modes:

- **Developer mode** reviews commits and working-tree changes in the current repository.
- **Reviewer mode** reviews a GitHub pull request identified by its number or URL.

This is a clean-room implementation inspired by multi-stage review workflows. Context collection, planning, review, result consolidation, and report generation are implemented entirely by this plugin.

## 2. Agent Compatibility

`review-pr` is a portable skill first and a platform package second. The Skill,
review criteria, and role definitions are the source of truth; each platform
manifest only describes how that platform discovers and installs them.

| Agent or harness | Packaging status | How to use it |
|---|---|---|
| Codex | Native | Add this repository as a Codex marketplace, install `review-pr`, then invoke `$review-pr` or make a matching natural-language request. |
| Claude Code | Native | Install or load the `review-pr` package, then invoke `/review-pr` or make a matching natural-language request. |
| Other coding agents | Portable skill; native package not yet provided | Import or expose `skills/review-pr/SKILL.md` using the agent's own skill mechanism, then use a natural-language request. |

The core workflow requires an agent that can read the repository and run
read-only Git commands. Reviewer mode additionally requires GitHub access
(normally through `gh`). Parallel role delegation improves coverage, but an
agent without subagents must report the affected review criteria as unavailable
rather than silently claiming an equivalent review.

When adding support for another agent, keep `skills/review-pr/`, `REVIEW.md`,
and the role definitions shared. Add only the agent-specific packaging or
adapter required for discovery, permissions, and subagent delegation. Document
the new support level in the table above instead of implying compatibility from
similarity alone.

## 3. Architecture

### Directory Structure

```text
review-pr/
├── .agents/
│   └── plugins/
│       └── marketplace.json
├── .codex-plugin/
│   └── plugin.json
├── .claude-plugin/
│   └── plugin.json
├── assets/
│   ├── review-pr-icon.png
│   └── review-workflow.svg
├── agents/
│   └── review/
│       ├── mechanical.md
│       ├── structural.md
│       └── contextual.md
├── skills/
│   └── review-pr/
│       ├── SKILL.md
│       └── checks/
│           ├── artifacts.md
│           ├── eligibility.md
│           └── scope.md
├── REVIEW.md
├── PRIVACY.md
├── README.md
├── TERMS.md
└── LICENSE
```

### Review Workflow

![Review PR workflow](assets/review-workflow.svg)

## 4. How to Use

### Codex

#### Step 1: Add the marketplace

Add this GitHub repository as a Codex plugin marketplace:

```bash
codex plugin marketplace add takuto-san/review-pr --ref main
```

You can verify that the marketplace is configured with:

```bash
codex plugin marketplace list
```

#### Step 2: Install the plugin

Install `review-pr` from the marketplace:

```bash
codex plugin add review-pr@review-pr
```

You can verify the installation with:

```bash
codex plugin list
```

For local development, validate the package with:

```bash
python3 /path/to/plugin-creator/scripts/validate_plugin.py /path/to/review-pr
```

#### Step 3: Open the target repository

Start a new Codex task in the repository you want to review.

To review a GitHub pull request, make sure GitHub CLI (`gh`) is installed and authenticated.

#### Step 4: Run the review

Review local changes:

```text
$review-pr
```

Review a GitHub pull request:

```text
$review-pr 123
```

Replace `123` with the pull-request number.

You can also use a natural-language request:

```text
Review my local changes
Review PR 123
Review this PR: https://github.com/owner/repository/pull/123
```

#### Step 5: Check the result

The plugin generates change-specific review criteria, gathers checks and evidence for each criterion, and returns one consolidated report.

If the report contains `Please Fix` findings, update the implementation and run `$review-pr` again.

#### Updating the plugin

Refresh the marketplace snapshot and reinstall the plugin:

```bash
codex plugin marketplace upgrade review-pr
codex plugin add review-pr@review-pr
```

Start a new Codex task after reinstalling so the updated skills are loaded into the new session.

### Claude Code

#### Step 1: Install or load the plugin

Load the plugin from the local repository:

```bash
claude --plugin-dir /path/to/review-pr
```

Validate the plugin if needed:

```bash
claude plugin validate /path/to/review-pr --strict
```

#### Step 2: Open the target repository

Start Claude Code in the repository you want to review.

To review a GitHub pull request, make sure GitHub CLI (`gh`) is installed and authenticated.

#### Step 3: Run the review

Review local changes:

```text
/review-pr
```

Review a GitHub pull request:

```text
/review-pr 123
```

Replace `123` with the pull-request number.

You can also use a natural-language request:

```text
Review my local changes
Review PR 123
Review this PR: https://github.com/owner/repository/pull/123
```

#### Step 4: Check the result

The plugin generates change-specific review criteria, gathers checks and evidence for each criterion, and returns one consolidated report.

If the report contains `Please Fix` findings, update the implementation and run `/review-pr` again.

### Requirements

The following environment and access are required to run the review:

- A coding agent that can load the skill, directly or through a native plugin
- A Git repository for Developer mode
- GitHub CLI (`gh`) installed and authenticated for Reviewer mode
- Access to the target repository and pull request
- Repository-defined test or analysis commands for mechanical verification
- Compatible read-only tools when external evidence is required

The review is read-only by default. It does not modify source files, install dependencies, change repository configuration, or post GitHub comments unless the user explicitly requests a separate action.

### Features

- Developer and Reviewer modes
- Review-need validation for pull requests
- Change-specific planning based on ISO/IEC 25010 quality characteristics
- Parallel mechanical, structural, and contextual evidence collection
- Criterion-centric consolidation instead of layer-centric reporting
- Mechanical checks mapped to the review criteria they materially verify
- Read-only context collection from explicitly referenced sources
- Compact review context instead of raw source documents
- Result consolidation, deduplication, and targeted evidence checks for `Please Fix` candidates
- Explicit checks, evidence, coverage, and limitations
