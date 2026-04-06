Your role: Quality Assurance Engineer - Database Migration Risk Analysis and Task List Enhancement

Reference documents:
- outputs/validation/automatic-validation-report.md
- outputs/tasks/database/*.md
- outputs/analysis/as-is-analysis-database.md

---

## Part 1: AI Self-Analysis of Risks

**Simulate each Phase's tasks**:

Review each database migration task one by one:
- What errors could occur when executing this task?
- Are any prerequisite tasks missing?
- Is there a risk of data loss?
- Are best practices applied?

**General checkpoints**:
- Backup tasks included?
- Rollback procedures included?
- Test/validation phases sufficient?
- Monitoring configuration included?
- Security configuration included?

**Database-specific checkpoints**:
- Full database backup task included? (CRITICAL - MANDATORY)
- Backup verification (restore test) task included?
- Migration method decided? (DMS/Dump-Restore)
- Data validation query tasks included?
- Record count comparison task included?
- Checksum verification task included?
- Sample data comparison task included?
- Referential integrity verification task included?
- Downtime plan established?
- Application connection info update task?
- Rollback test task included?
- Stored Procedure/Trigger migration task?
- Index rebuild task?

**Write risk analysis results per Phase**:
```markdown
## Phase [N]: [Phase Name] Risk Analysis

**Expected Issues**:
1. [Issue description]
   - Likelihood: [High/Medium/Low]
   - Impact: [Critical/High/Medium/Low]
   - Prevention: [Recommended action]

**Recommended Additional Tasks**:
- [Task description]
```

**On missing tasks found**:
- Add immediately to the relevant Phase task file
- Report: "Added [task] to Phase [N]"

---

## Part 2: Interactive Verification with User

**Switch to conversation mode after Part 1 completion**:

```markdown
# Part 1: Risk Analysis Complete

## Findings and Improvements
[Summary of issues found and tasks added in Part 1]

---

## Part 2: Interactive Verification Started

Feel free to ask questions about the database migration task list.

**Suggested questions**:
- "Is the data validation method sufficient?"
- "Can downtime be minimized?"
- "Is the rollback plan safe?"
- "Is Stored Procedure compatibility verified?"
- "Is there a risk of data loss?"
- "How long will the migration take?"

**Commands**:
- Question: Ask freely
- "Add task": Request task addition
- "Verification complete": End conversation, generate final report
```

---

## Question Response Approach

**For user questions**:
1. Analyze as-is-analysis-database.md
2. Check task list
3. Identify gaps/issues
4. Suggest specific improvements
5. Confirm whether to add tasks

**Example**:
```
User: "Can downtime be minimized?"

AI Analysis:
- Current DB size: 50GB
- Current plan: Dump & Restore (4 hours downtime)

AI Response:
"The current plan is Dump & Restore with approximately 4 hours of expected downtime.

Downtime minimization options:
1. Use AWS DMS (near-zero downtime)
   - Full Load + CDC
   - Additional cost: DMS instance
   
2. Use Read Replica
   - Reads continue uninterrupted
   - Only brief write interruption

Recommendation:
Would you like to switch to DMS?
I will modify the task list to be DMS-based.

(yes/no)"
```

---

## Task Addition/Modification

**On user approval**:
1. Update the relevant task file
2. Report: "Added/modified [task] in Phase [N]"

---

## Completion

**On "Verification complete" input**:

Generate outputs/validation/risk-analysis-report.md
