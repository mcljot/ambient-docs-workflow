# /resume

Read `.claude/skills/controller/SKILL.md` and follow it.

Resume an interrupted documentation workflow. Load existing state and continue from the last completed stage.

## Argument Parsing

Parse the following from `$ARGUMENTS`:

- **TICKET** (required): JIRA ticket identifier (e.g., `RHAISTRAT-123`)
- **--pr <url>**: Additional PR/MR URL to include (can appear multiple times)
- **--integrate**: Enable the integration stage (can be added on resume)
- **--create-jira <PROJECT>**: Enable JIRA ticket creation (can be added on resume)

If no ticket is provided, STOP and ask the user to provide one.

## Load State

```bash
CLAUDE_DOCS_DIR="${PWD}/.claude/docs"
SAFE_TICKET=$(echo "$TICKET" | tr '[:upper:]' '[:lower:]' | tr '-' '_')
STATE_FILE="${CLAUDE_DOCS_DIR}/workflow/workflow_${SAFE_TICKET}.json"
```

If no state file exists, inform the user and suggest using `/start` instead.

If the state file exists:
1. Add any new `--pr` URLs to `options.pr_urls` (deduplicate)
2. Set `options.integrate = true` if `--integrate` was provided
3. Set `options.create_jira_project` if `--create-jira` was provided

## Execution

After loading state, follow the controller's stage execution logic:
1. Determine the next incomplete stage
2. If all stages are completed, display the final status and stop
3. Otherwise, read the stage skill and execute it
4. Continue through remaining stages

$ARGUMENTS
