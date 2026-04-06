---
name: change-management
description: Change classification (Standard/Normal/Critical), rollback procedures (immediate/standard/complex), audit.md-based change tracking, risk level assessment per Stage. Reference for change management across all Stages.
---
# Change Management for AI-Driven Modernization

## Purpose
A lightweight framework for safely managing and tracking changes in AI-driven modernization projects.

---

## Change Classification

### Standard Changes (No Approval Required)
- Code generation and modification
- Container image builds
- Dev/Test environment deployments
- Test execution
- Document generation

### Normal Changes (User Approval Required)
- Infrastructure provisioning (VPC, ECS/EKS, RDS)
- Staging environment deployment
- Configuration changes (security groups, IAM)
- DB schema changes

### Critical Changes (Explicit Approval + Rollback Plan Required)
- Production deployment
- DB migration cutover
- Traffic switching
- Resource deletion
- Irreversible changes

---

## Rollback Procedures

### Rollback Triggers
- Error rate > 5%
- Response time P95 > 2000ms
- Health check failures > 10%
- Data inconsistency detected
- User request

### Rollback Types

**Immediate Rollback (< 5 min)**:
- Traffic switching: Execute `rollback-to-blue.sh`
- Container deployment: Rollback to previous image tag
- IaC: Restore to previous state (`cdk deploy` previous version)

**Standard Rollback (5~30 min)**:
- Restore DB connection information
- Revert infrastructure changes
- Restore configuration

**Complex Rollback (> 30 min)**:
- Restore from DB backup
- Full infrastructure rebuild
- Proceed after consultation with user

---

## Change Tracking

### audit.md-Based Tracking
All changes are recorded in `outputs/audit.md`.

**Items to Record**:
- Change content and reason
- Execution time (timestamp)
- Impact scope
- Approval status and approver
- Result (success/failed/rolled back)
- List of created/modified files

### Change Record Format
```markdown
## [Phase/Task Name]
**Timestamp**: [ISO timestamp]
**Type**: [Standard/Normal/Critical]
**Action**: [Change performed]
**Files Changed**: [File list]
**Result**: [Success/Failed/Rolled Back]
**Approval**: [Auto/User approval timestamp]
**Notes**: [Special notes]
```

---

## Risk Assessment (Simplified)

### Risk Level Determination

| Level | Criteria | Response |
|-------|----------|----------|
| Low | Dev/Test environment, easily reversible | Proceed then report |
| Medium | Staging environment, infrastructure changes | Proceed after user approval |
| High | Production, DB, traffic switching | Approval + rollback plan + dry-run |

### Dry-Run Required For
- `cdk diff` / `terraform plan` → Before infrastructure changes
- Traffic switching scripts → Verify before execution
- DB migration → Execute in test environment first

---

## Change Management by Stage

### Stage 1~2 (Analysis)
- Change type: Document creation only
- Risk: Low
- Approval: Not required (analysis result review only)

### Stage 3 (Planning)
- Change type: Task list creation
- Risk: Low
- Approval: Plan review and approval

### Stage 4 (Validation)
- Change type: Tool access testing, task list refinement
- Risk: Low
- Approval: Validation result confirmation

### Stage 5 (Execution)
- Change type: Code generation, infrastructure deployment, containerization
- Risk: Medium~High
- Approval: Phase-level approval gates

### Stage 5 (Testing/QA/Traffic Cutover)
- Change type: Production traffic switching
- Risk: High
- Approval: Explicit approval per Phase, rollback scripts required
