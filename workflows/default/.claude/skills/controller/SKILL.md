---
name: controller
description: Top-level workflow controller that manages stage transitions for the documentation pipeline.
---

# Documentation Workflow Controller

You are the workflow controller. Your job is to manage the documentation pipeline by
executing stages sequentially and handling transitions between them.

## Stages

1. **Requirements** — `.claude/skills/requirements/SKILL.md`
   Analyze JIRA ticket, PRs, and specs to extract documentation requirements.

2. **Planning** — `.claude/skills/planning/SKILL.md`
   Create a structured documentation plan using JTBD framework.

3. **Writing** — `.claude/skills/writing/SKILL.md`
   Write complete AsciiDoc or MkDocs documentation from the plan.

4. **Technical Review** — `.claude/skills/technical-review/SKILL.md`
   Validate technical accuracy of code examples, prerequisites, and commands.
   Iterates up to 3 times if confidence is not HIGH.

5. **Style Review** — `.claude/skills/style-review/SKILL.md`
   Apply style guide checks (IBM SG, Red Hat SSG) and edit files in place.

6. **Integration** *(optional, requires `--integrate`)* — `.claude/skills/integrate/SKILL.md`
   Detect the repo's build framework and integrate drafts. Two-phase: PLAN then EXECUTE with user confirmation.

7. **Create JIRA** *(optional, requires `--create-jira <PROJECT>`)* — `.claude/skills/create-jira/SKILL.md`
   Create a documentation JIRA ticket linked to the parent ticket.

## State Management

### State File Location

```
.claude/docs/workflow/workflow_<SAFE_TICKET>.json
```

Where `<SAFE_TICKET>` is the ticket ID lowercased with hyphens replaced by underscores.

### State File Structure

```json
{
  "ticket": "TICKET-123",
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z",
  "current_stage": "requirements",
  "status": "pending",
  "options": {
    "pr_urls": [],
    "format": "adoc",
    "integrate": false,
    "create_jira_project": null
  },
  "stages": {
    "requirements": {"status": "pending", "output_file": null, "started_at": null, "completed_at": null},
    "planning": {"status": "pending", "output_file": null, "started_at": null, "completed_at": null},
    "writing": {"status": "pending", "output_file": null, "started_at": null, "completed_at": null},
    "technical_review": {"status": "pending", "output_file": null, "started_at": null, "completed_at": null, "iterations": 0},
    "review": {"status": "pending", "output_file": null, "started_at": null, "completed_at": null},
    "integrate": {"status": "pending", "output_file": null, "started_at": null, "completed_at": null, "phase": null},
    "create_jira": {"status": "pending", "output_file": null, "started_at": null, "completed_at": null}
  }
}
```

### State Update Commands

**Mark stage as in_progress:**

```bash
TMP=$(mktemp)
jq --arg stage "<STAGE>" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.stages[$stage].status = "in_progress" | .stages[$stage].started_at = $now | .updated_at = $now | .status = "in_progress" | .current_stage = $stage' \
   "$STATE_FILE" > "$TMP"
mv "$TMP" "$STATE_FILE"
```

**Mark stage as completed:**

```bash
TMP=$(mktemp)
jq --arg stage "<STAGE>" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" --arg output "$OUTPUT_FILE" \
   '.stages[$stage].status = "completed" | .stages[$stage].completed_at = $now | .stages[$stage].output_file = $output | .updated_at = $now' \
   "$STATE_FILE" > "$TMP"
mv "$TMP" "$STATE_FILE"
```

**Mark stage as failed:**

```bash
TMP=$(mktemp)
jq --arg stage "<STAGE>" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.stages[$stage].status = "failed" | .updated_at = $now | .status = "failed"' \
   "$STATE_FILE" > "$TMP"
mv "$TMP" "$STATE_FILE"
```

Replace `<STAGE>` with the actual stage name (e.g., `requirements`, `planning`).

## How to Execute a Stage

1. **Announce** the stage to the user before doing anything else:
   "Starting Stage N: {Stage Name} (dispatched by `.claude/skills/controller/SKILL.md`)."
2. **Update state** to `in_progress` using the bash command above
3. **Read** the stage skill file from the paths listed above
4. **Execute** the skill's instructions directly — the user should see your progress
5. **Verify** the output file exists
6. **Update state** to `completed` with the output file path
7. **Report** results to the user with a brief summary
8. **Proceed** to the next stage automatically (unlike the bugfix workflow, this pipeline auto-advances)

### Auto-Advance vs. Pause

This workflow **auto-advances** through stages by default. The only pauses are:

- **Integration stage**: MUST pause for user confirmation between PLAN and EXECUTE phases
- **Access failures**: MUST stop immediately and report the error
- **Technical review iteration**: Automatically iterates (no user pause needed)

## Determining the Next Stage

The active stages depend on the options:

```
ALWAYS: requirements → planning → writing → technical_review → review
IF --integrate: → integrate
IF --create-jira: → create_jira
```

Find the first stage that is not `completed` and execute it. If all applicable stages are completed, the workflow is done.

```bash
INTEGRATE_OPT=$(jq -r '.options.integrate // false' "$STATE_FILE")
CREATE_JIRA_PROJ=$(jq -r '.options.create_jira_project // ""' "$STATE_FILE")
STAGES="requirements planning writing technical_review review"
case "$INTEGRATE_OPT" in
    "true") STAGES="$STAGES integrate" ;;
esac
case "$CREATE_JIRA_PROJ" in
    ""|"null") ;;
    *) STAGES="$STAGES create_jira" ;;
esac

NEXT_STAGE=""
for STAGE in $STAGES; do
    STAGE_STATUS=$(jq -r ".stages.${STAGE}.status" "$STATE_FILE")
    case "$STAGE_STATUS" in
        "completed") ;;
        *) NEXT_STAGE="$STAGE"; break ;;
    esac
done
```

## Technical Review Iteration Loop

The technical review stage has special iteration logic:

1. Execute the technical review skill
2. Increment the iteration counter in state:
   ```bash
   TMP=$(mktemp)
   jq '.stages.technical_review.iterations += 1' "$STATE_FILE" > "$TMP"
   mv "$TMP" "$STATE_FILE"
   ```
3. Read the review report and check the **Overall technical confidence** rating
4. If confidence is **HIGH**: mark `technical_review` as completed, proceed to style review
5. If confidence is **MEDIUM** or **LOW** and iterations < 3:
   - Read the writing skill
   - Fix the critical and significant issues identified in the review
   - Re-run the technical review
6. If confidence is **MEDIUM** or **LOW** and iterations >= 3:
   - Mark the stage as completed with a note that manual review is recommended
   - Proceed to style review

## Integration Phase Dispatch

The integration stage uses a `phase` field in state for conditional dispatch:

- `null` (first entry): Execute PLAN phase, set phase to `awaiting_confirmation`
- `awaiting_confirmation`: Present plan summary, ask user to confirm (yes/no)
  - If YES: set phase to `confirmed`, execute EXECUTE phase
  - If NO: set phase to `declined`, mark as completed with plan file as output
- `confirmed`: Execute EXECUTE phase, mark as completed
- `declined`: Mark as completed, inform user plan is saved for manual reference

## Workflow Completion

After all applicable stages complete:

```bash
TMP=$(mktemp)
jq --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.status = "completed" | .updated_at = $now' \
   "$STATE_FILE" > "$TMP"
mv "$TMP" "$STATE_FILE"
```

Display the final status summary showing:
- All stages with completion status
- Output file locations
- Created JIRA ticket URL (if applicable)
- Total time elapsed

## Access Failure Protocol

If any stage fails due to access issues (JIRA, GitHub, GitLab):

1. **STOP IMMEDIATELY** — Do not proceed to the next stage
2. **Report the exact error** — Display the full error message
3. **List available env files** — Run `ls -la ~/.env* 2>/dev/null`
4. **Mark the stage as failed** in the state file
5. **Inform the user** they can fix credentials and resume with `/resume <TICKET>`
6. **NEVER guess or infer** — No assumptions about ticket or PR content

## JIRA Environment File Fallback

If JIRA access fails during a stage, try alternate env files before giving up:

1. Search for `~/.env*` files containing "jira": `ls -la ~/.env*jira* ~/.env*.jira* 2>/dev/null`
2. Source each alternate file and retry
3. If all fail, reset to `~/.env` and retry
4. If still failing, STOP and report the error

## Rules

- **Auto-advance through stages** — do not wait between stages unless integration confirmation is needed
- **Always update state** — before starting a stage and after completing it
- **Verify output files** — after each stage, confirm the expected output file exists
- **Sequential execution only** — never run stages in parallel
- **Announce each stage** — so the user knows the workflow is progressing
