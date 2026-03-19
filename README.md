# Documentation Workflow

Multi-stage documentation pipeline that transforms JIRA tickets into complete, reviewed technical documentation. Produces AsciiDoc (default) or Material for MkDocs Markdown output.

## Overview

This workflow orchestrates a sequential seven-stage pipeline, each building on the previous stage's output:

### 1. Requirements Analysis

Extracts documentation needs from JIRA tickets, PRs/MRs, and engineering specs. Traverses ticket relationships, analyzes code changes, and produces a structured requirements document with prioritized recommendations.

### 2. Planning

Creates a comprehensive documentation plan using the Jobs to Be Done (JTBD) framework. Defines module specifications (type, title, audience, content points), maps content to user journey phases, and determines implementation order.

### 3. Writing

Writes complete, production-ready documentation following the plan. Supports two output formats:

- **AsciiDoc**: Concept, procedure, and reference modules with assemblies following Red Hat modular documentation standards
- **MkDocs Markdown**: Pages with YAML frontmatter and Material for MkDocs conventions

### 4. Technical Review (iterative)

Validates documentation for technical accuracy using developer and architect lenses. Checks code examples, prerequisites, commands, failure paths, and architectural coherence. Iterates up to 3 times with the writer if confidence is not HIGH.

### 5. Style Review

Applies IBM Style Guide and Red Hat Supplementary Style Guide checks. Edits files in place to fix clear violations and reports remaining issues requiring manual review.

### 6. Integration (optional)

Detects the repository's documentation build framework (Antora, ccutil, MkDocs, Sphinx, etc.) and integrates drafts. Operates in two phases: PLAN (propose changes) then EXECUTE (apply changes after user confirmation).

### 7. JIRA Creation (optional)

Creates a documentation JIRA ticket linked to the parent engineering ticket. Checks for existing documentation links to prevent duplicates. Adapts behavior based on project visibility (public vs. private).

## Getting Started

### Prerequisites

- `JIRA_AUTH_TOKEN` in environment (required)
- `JIRA_EMAIL` in environment (required for JIRA Cloud)
- `GITHUB_TOKEN` and/or `GITLAB_TOKEN` for PR/MR access (optional)
- `jq` for JSON processing
- `python3` for script utilities

### Usage

Start a new workflow:

```
/start RHAISTRAT-123
```

Start with a related PR:

```
/start RHAISTRAT-123 --pr https://github.com/org/repo/pull/456
```

Start with MkDocs output:

```
/start RHAISTRAT-123 --mkdocs
```

Start with integration and JIRA creation:

```
/start RHAISTRAT-123 --integrate --create-jira INFERENG
```

Check status:

```
/status RHAISTRAT-123
```

Resume an interrupted workflow:

```
/resume RHAISTRAT-123
```

Add options on resume:

```
/resume RHAISTRAT-123 --pr https://github.com/org/repo/pull/789 --integrate
```

## Output

All workflow outputs are organized under `.claude/docs/`:

```
.claude/docs/
├── workflow/           # State files (JSON) for resume capability
├── requirements/       # Stage 1 output
├── plans/              # Stage 2 output
└── drafts/             # Stage 3-6 output (per-ticket folders)
    └── <ticket>/
        ├── _index.md
        ├── _technical_review.md
        ├── _review_report.md
        ├── assembly_*.adoc      # (AsciiDoc format)
        └── modules/
            ├── con_*.adoc
            ├── proc_*.adoc
            └── ref_*.adoc
```

## Configuration

This workflow is configured via `.ambient/ambient.json`. The behavioral guidelines are in `CLAUDE.md`.

The workflow controller is at `.claude/skills/controller/SKILL.md` and manages stage transitions. Individual stage skills are at `.claude/skills/{stage-name}/SKILL.md`.

## Options Reference

| Option | Description |
|--------|-------------|
| `--pr <url>` | GitHub PR or GitLab MR URL to include in analysis (repeatable) |
| `--mkdocs` | Output Material for MkDocs Markdown instead of AsciiDoc |
| `--integrate` | Enable build framework integration stage |
| `--create-jira <PROJECT>` | Create a docs JIRA ticket in the specified project |

## Based On

This workflow is a hosted version of the [`docs-workflow`](https://github.com/redhat-documentation/redhat-docs-agent-tools) command from the `redhat-docs-agent-tools` project, adapted for the Ambient Code Platform.
