# AI Execution Guidelines

## Purpose
Define rules and patterns for AI to directly execute modernization tasks.

## Language Rule
**MANDATORY**: All user-facing messages (checkpoints, approvals, errors, escalations, audit logs) must follow the language rule defined in `steering/workflow-orchestration.md` > Language Detection and Adaptation. The message templates below are in English as structural references — generate actual messages in the user's detected language.

## Core Principle

**AI-Driven Execution**: AI is a direct executor, not an advisor.

```
Traditional:               AI-Driven:
AI: "Run this command"     →  AI: "Executing this command..."
User: [executes]           →  AI: [executes via tools]
User: "Done"               →  AI: "✅ Complete. Please review."
```

---

## AI Execution Responsibilities

### 1. Code Generation
- Dockerfile, IaC code (CDK/Terraform), configuration files, test scripts

### 2. Command Execution
- Docker build/test, IaC deployment, DB migration, test execution, AWS service configuration

### 3. Result Verification
- Exit code checks, resource creation confirmation, health checks, validation queries, metric monitoring

### 4. Error Handling
- Detect failure → Analyze error → Attempt auto-fix → Retry → Escalate to user

### 5. Progress Reporting
- Log all tasks in audit.md, update checkboxes, report results, request approval at checkpoints

---

## Audit Trail (audit.md)

### Location
`outputs/audit.md`

### What to Record
- Start/completion of all stages (Stage 1~6) with timestamps
- **User prompt input (VERBATIM)**: Record the user's original text exactly as written at stage start, key decision points (approvals, rejections, direction changes), and when answering questions. Do not summarize, paraphrase, or extract keywords. Use `> ` blockquote format.
- **User answers to questions (VERBATIM)**: When user responds to plan.md questions or AI-posed questions, record the full answer in `> ` blockquote format without any modification.
- Decisions and approval records
- File creation/modification logs
- Error occurrence and resolution process
- Reason when tests are skipped

### Format
```markdown
## [Stage/Phase Name]
**Timestamp**: [ISO timestamp]
**User Prompt** (verbatim):
> [Paste user's original input exactly as written]
**Action**: [Action performed]
**User Answer/Decision** (verbatim):
> [Paste user's answer exactly as written]
**Result**: [Result]
**Files**: [List of created/modified files]
**Notes**: [Special notes]
```

---

## Proactive Execution Patterns

### Suggested Questions Pattern
At the start of each stage, AI presents key questions before waiting for user input:
```
Before we begin [Stage Name], here are the key items to confirm:
1. [Key question 1]
2. [Key question 2]
3. [Key question 3]

Do you have this information ready, or should we go through them one by one?
```

### Missing Item Detection Pattern
After completing analysis or plan generation, AI automatically checks for gaps:
```
I've completed the [analysis/plan]. Let me check for potential gaps:
- ✅ [Item covered]
- ✅ [Item covered]
- ⚠️ [Item not covered] — Is this intentional, or should we address it?
```

### Environment-Specific Guidance Pattern
When AI detects specific environment characteristics, proactively suggest relevant considerations:
- Production environment detected → Suggest cutover strategy, rollback plan
- Worker/queue pattern detected → Ask about graceful shutdown, task duration
- External integrations detected → Ask about authentication, network paths
- Security constraints detected → Suggest compliance considerations

### Checkbox Verification Pattern (MANDATORY)
After completing each task, AI must verify the checkbox was updated:
```
After task completion:
1. Update checkbox [ ] → [x] in the task file IMMEDIATELY
2. Do NOT proceed to next task until checkbox is updated
```
At the end of each Phase, AI must verify all checkboxes:
```
Phase completion check:
1. Read the task file for the completed Phase
2. Count [ ] (incomplete) vs [x] (complete)
3. If any [ ] remains → WARNING: "Task [name] is not marked complete. Updating now."
4. Fix any missed checkboxes before reporting Phase completion
```

### Step Completion Verification (MANDATORY)
AI must verify all sub-steps are completed before moving to the next Stage or Phase.

**At each Stage/sub-step start**:
```
AI announces: "Items to perform in this step:"
1. [Sub-step 1]
2. [Sub-step 2]
3. [Sub-step 3]
```

**At each Stage/sub-step completion**:
```
AI verifies:
✅ [Sub-step 1] — Complete
✅ [Sub-step 2] — Complete
⬜ [Sub-step 3] — Incomplete → This item must be completed first

💡 Useful questions to check for incomplete items:
- "[relevant question 1]"
- "[relevant question 2]"
```

**Rules**:
- Do NOT proceed to the next Stage until all sub-steps of the current Stage are verified complete
- If a sub-step is intentionally skipped, it must follow the Anti-Skip Rule (see `workflow-orchestration.md`)
- Always provide question suggestions for incomplete or skipped items to help the user engage

---

## Stage Transition Protocol

### On Stage Completion
1. Verify all output files for the current Stage are generated
2. Log completion in `outputs/audit.md` with timestamp
3. Propose next Stage with `/clear` recommendation (optional, with reason)
4. Wait for user confirmation before proceeding

### After `/clear` (New Session Resume)
1. Scan `outputs/` directory to detect current progress
2. Report detected state: "Stages up to N are complete."
3. Load the next Stage's prompt file from `prompts/` automatically
4. Propose to proceed: "Shall we start Stage N+1?"

---

## Execution Patterns

### Standard Execution Flow

Follow this sequence for each task:

1. **Prepare**: Load context, verify prerequisites, log start in audit.md
2. **Create**: Generate code/config files, save to appropriate location
3. **Execute**: Run build/deploy commands, check exit codes, log execution in audit.md
4. **Verify**: Confirm expected results, run verification checks, collect evidence
5. **Handle Errors**: Analyze errors, attempt auto-fix, retry, escalate if unresolvable
6. **Report**: Present results, update checkboxes, request approval

---

## Checkpoint Strategy

### Approval Required (Major Checkpoint)
- After each Phase completion
- Before production deployment
- Before DB migration cutover
- Before irreversible changes

### Report Only (Minor Checkpoint)
- After code generation
- After artifact build
- After test execution
- After Dev/Staging deployment

### Approval Message Format
```markdown
# 🔍 Checkpoint: [Phase Name]

## Completed Work
[Summary]

## Results
[Key results and metrics]

## Next Steps
[Next task content]

## Risk Assessment
- Risk Level: [Low/Medium/High]
- Reversible: [Yes/No]
- Rollback Plan: [Available/Not needed]

Type 'APPROVED' to continue.
```

---

## Error Handling

### Auto-Handling Flow
1. **Detect**: Command failure or verification error detected
2. **Analyze**: Analyze error message and context
3. **Attempt Fix**: Auto-remediation (max 2 attempts)
4. **Retry**: Re-execute after fix
5. **Escalate**: Report to user if unresolvable

### Escalation Format
```markdown
# ⚠️ User Input Required

## Issue
[Description]

## Attempted Solutions
1. [Attempt 1] - Failed
2. [Attempt 2] - Failed

## Options
A) [Option 1]
B) [Option 2]
C) Skip this step and continue
D) Rollback and try different approach

[Answer]:
```

---

## Safety Measures

### Auto-Rollback
Implement auto-rollback for failed deployments, DB migrations, and infrastructure changes.

### Dry-Run Mode
Before risky operations:
1. Run dry-run first (`cdk diff`, `terraform plan`)
2. Display changes
3. Request approval
4. Execute actual operation

---

## Mandatory Test Execution

### Container Tests (MANDATORY)
- Local container test: Cannot be skipped
- Vulnerability scan: Cannot be skipped
- Approval gate required before ECR push
- On test failure: Fix and retry, escalate if unresolvable

### Test Skip Rules
- ✅ Must attempt test first
- ✅ If unable to execute (e.g., external DB required): Explain reason and get approval
- ✅ Document skip reason in audit.md
- ✅ Plan testing in next environment (AWS dev/staging)
- ❌ Never silently skip tests

### Test Skip Documentation Format
```markdown
## Test Execution Notes
- **Status**: ⚠️ Partially Skipped
- **Reason**: [Reason]
- **Tested Items**: [List]
- **Skipped Items**: [List]
- **Alternative**: [Test plan for next environment]
- **User Approval**: [Timestamp]
```

---

## Approval Rules Summary

### Approval Required
- ✅ Production deployment
- ✅ DB migration cutover
- ✅ Resource deletion
- ✅ Irreversible changes
- ✅ Expected cost > $100/month

### Can Proceed Without Approval
- ✅ Code generation
- ✅ Container build
- ✅ Dev/Test deployment
- ✅ Test execution
- ✅ Document generation

### Rollback Must Be Implemented
- ✅ Failed deployments
- ✅ Failed migrations
- ✅ Failed infrastructure changes
