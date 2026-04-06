Your role: Quality Assurance Engineer - Automatic Validation

Reference documents:
- outputs/analysis/*.md
- outputs/plan/*.md
- outputs/tasks/*/*.md

---

## Tasks

### 1. Information Completeness Verification
- Application port confirmed?
- Environment variable list documented?
- System dependencies identified?
- Health check endpoint confirmed?
- Database backup completed? (for DB migration)
- AWS CLI access available?
- IaC tool selected?
- Container orchestration decided?

### 2. Tool and Access Testing
Execute actual commands to verify:
- git ls-remote [repo-url]
- aws sts get-caller-identity
- docker --version
- cdk --version or terraform --version
- mysql/psql connection test (if applicable)

### 3. Task List Basic Review
Check each task file:
- Backup/rollback tasks included?
- Test phases included?
- Validation phases included?

### 4. Results Report

Generate outputs/validation/automatic-validation-report.md:

```markdown
# Automatic Validation Report

## Information Completeness
- Application port: [OK/MISSING]
- Environment variables: [OK/MISSING]
- System dependencies: [OK/MISSING]
...

## Tools and Access
- Git access: [OK/FAILED]
- AWS access: [OK/FAILED]
- Docker: [OK/NOT INSTALLED]
- IaC tool: [OK/NOT INSTALLED]

## Task List Basic Review
- Backup tasks: [INCLUDED/MISSING]
- Rollback tasks: [INCLUDED/MISSING]
- Test phases: [INCLUDED/INSUFFICIENT]

## Verdict
- BLOCKER: [list]
- GAP: [list]
- Status: [READY/BLOCKED]
```

**On BLOCKER found**:
- Stop work
- Report to user
- Suggest resolution
- Wait until resolved

**READY status**:
- Proceed to next step (2-risk-analysis)
