# /start

Read `.claude/skills/controller/SKILL.md` and follow it.

Start a new documentation workflow. Parse arguments, validate access tokens, initialize state, and begin sequential stage execution.

## Argument Parsing

Parse the following from `$ARGUMENTS`:

- **TICKET** (required): JIRA ticket identifier (e.g., `RHAISTRAT-123`)
- **--pr <url>**: GitHub PR or GitLab MR URL (can appear multiple times)
- **--mkdocs**: Output Material for MkDocs Markdown instead of AsciiDoc
- **--integrate**: Enable the integration stage
- **--create-jira <PROJECT>**: Enable JIRA ticket creation in the specified project

If no ticket is provided, STOP and ask the user to provide one.

## Pre-flight Validation

Validate access tokens before proceeding:

```bash
echo "Validating access tokens..."
HAS_ERRORS=false

if [[ -z "${JIRA_AUTH_TOKEN:-}" ]]; then
    echo "ERROR: JIRA_AUTH_TOKEN is not set."
    echo "  Add JIRA_AUTH_TOKEN to ~/.env and source it."
    HAS_ERRORS=true
else
    echo "  JIRA_AUTH_TOKEN: configured"
fi

if [[ -n "${GITHUB_TOKEN:-}" ]]; then
    echo "  GITHUB_TOKEN: configured"
else
    echo "  GITHUB_TOKEN: not set (required if using GitHub PRs)"
fi

if [[ -n "${GITLAB_TOKEN:-}" ]]; then
    echo "  GITLAB_TOKEN: configured"
else
    echo "  GITLAB_TOKEN: not set (required if using GitLab MRs)"
fi

if [[ "$HAS_ERRORS" == "true" ]]; then
    echo ""
    echo "CRITICAL: Required access tokens are missing."
    echo "The workflow WILL NOT proceed without JIRA_AUTH_TOKEN."
    echo ""
    echo "Available env files:"
    ls -la ~/.env* 2>/dev/null || echo "  No ~/.env* files found"
    exit 1
fi
```

**If JIRA_AUTH_TOKEN is missing, STOP IMMEDIATELY.** Do not proceed.

## State Initialization

```bash
CLAUDE_DOCS_DIR="${PWD}/.claude/docs"
SAFE_TICKET=$(echo "$TICKET" | tr '[:upper:]' '[:lower:]' | tr '-' '_')
STATE_FILE="${CLAUDE_DOCS_DIR}/workflow/workflow_${SAFE_TICKET}.json"

mkdir -p "${CLAUDE_DOCS_DIR}/workflow" "${CLAUDE_DOCS_DIR}/requirements" "${CLAUDE_DOCS_DIR}/plans" "${CLAUDE_DOCS_DIR}/drafts"
```

If a state file already exists for this ticket, treat it as a resume — load existing state and continue from the next incomplete stage.

Otherwise, create a new state file following the structure defined in the controller skill.

## Execution

After initialization, follow the controller's stage execution logic:
1. Determine the next incomplete stage
2. Read the stage skill
3. Execute it
4. Update state
5. Proceed to the next stage

$ARGUMENTS
