# /status

Display the current status of a documentation workflow. Do NOT run any stages.

## Argument Parsing

Parse the TICKET from `$ARGUMENTS`. If no ticket is provided, STOP and ask the user to provide one.

## Display Status

```bash
CLAUDE_DOCS_DIR="${PWD}/.claude/docs"
SAFE_TICKET=$(echo "$TICKET" | tr '[:upper:]' '[:lower:]' | tr '-' '_')
STATE_FILE="${CLAUDE_DOCS_DIR}/workflow/workflow_${SAFE_TICKET}.json"
```

If no state file exists, inform the user that no workflow was found for this ticket.

Otherwise, read the state file and display:

```
Documentation Workflow Status: <TICKET>

Overall Status: <status>
Current Stage:  <current_stage>
PR/MR URLs:     <pr_urls>
Format:         <format>

Stages:
  [x] requirements     -> <output_file>
  [x] planning         -> <output_file>
  [>] writing
  [ ] technical_review
  [ ] review
  [ ] integrate        (--integrate)
  [ ] create_jira      (--create-jira <PROJECT>)
```

Use `[x]` for completed, `[>]` for in_progress, `[!]` for failed, `[ ]` for pending.
Only show integrate and create_jira stages if their respective options are enabled.

**After displaying status, STOP. Do not run any stages.**

$ARGUMENTS
