# Documentation Workflow

Multi-stage documentation pipeline through these stages:

1. **Requirements** (`/start`) — Analyze JIRA ticket, PRs, and specs for documentation needs
2. **Planning** — Create documentation plan with JTBD framework and module specifications
3. **Writing** — Generate complete AsciiDoc or MkDocs documentation
4. **Technical Review** — Validate code examples, prerequisites, commands (iterative, up to 3 rounds)
5. **Style Review** — Apply style guide checks and edit files in place
6. **Integration** (`--integrate`) — *(Optional)* Integrate drafts into repo's build framework
7. **JIRA Creation** (`--create-jira <PROJECT>`) — *(Optional)* Create a linked documentation ticket

The workflow controller lives at `.claude/skills/controller/SKILL.md`.
It defines how to execute stages, manage state, and handle transitions.
Stage skills are at `.claude/skills/{name}/SKILL.md`.
State files go in `.claude/docs/workflow/`.
Output goes in `.claude/docs/drafts/`.

## Principles

- **Never guess or infer content.** If JIRA access fails, STOP. Do not fabricate ticket content.
- **Access validation first.** Validate JIRA_AUTH_TOKEN before any work. Check GitHub/GitLab tokens before PR analysis.
- **State-driven execution.** Always update the JSON state file before and after each stage. This enables resume.
- **Sequential stages.** Each stage depends on the previous stage's output. Never run stages in parallel.
- **In-place editing.** The style review stage edits draft files directly — no copies to separate folders.
- **User confirmation for integration.** The integrate stage presents a plan and waits for explicit approval before making changes.
- **Show progress.** Announce each stage before executing it. Show completion status after each stage.

## Hard Limits

- No proceeding without JIRA_AUTH_TOKEN — the workflow halts immediately
- No fabricating ticket content — if access fails, stop and report the error
- No skipping the technical review iteration protocol — always check confidence and iterate if needed
- No attaching docs plans to public JIRA projects — check project visibility first
- No creating duplicate JIRA tickets — check for existing "is documented by" links

## Output Formats

### AsciiDoc (default)

```
.claude/docs/drafts/<ticket>/
├── _index.md
├── _review_report.md
├── _technical_review.md
├── assembly_*.adoc
└── modules/
    ├── <concept>.adoc
    ├── <procedure>.adoc
    └── <reference>.adoc
```

### MkDocs Markdown (`--mkdocs`)

```
.claude/docs/drafts/<ticket>/
├── _index.md
├── _review_report.md
├── _technical_review.md
├── mkdocs-nav.yml
└── docs/
    ├── <concept>.md
    ├── <procedure>.md
    └── <reference>.md
```

## Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `JIRA_AUTH_TOKEN` | Yes | JIRA API authentication |
| `JIRA_EMAIL` | Yes | JIRA Cloud authentication |
| `JIRA_URL` | No | Defaults to `https://redhat.atlassian.net` |
| `GITHUB_TOKEN` | No | GitHub PR access |
| `GITLAB_TOKEN` | No | GitLab MR access |
