# Readiness Checklist

## Purpose
Verify all prerequisites are met before starting modernization tasks.

---

## Container Readiness

### Application Information
- [ ] Application port identified
- [ ] Environment variables documented
- [ ] System dependencies known
- [ ] Health check endpoint confirmed or will create
- [ ] Startup time estimated

### Container Decisions
- [ ] Container orchestration decided (ECS/EKS)
- [ ] Base image preference decided
- [ ] Multi-stage build planned

### Severity if Missing
- Port unknown: **GAP** (will ask during containerization)
- Orchestration not decided: **BLOCKER** (must decide before IaC)
- Env vars unknown: **GAP** (will ask during containerization)

---

## Infrastructure Readiness

### AWS Access
- [ ] AWS CLI access confirmed and working
- [ ] Target AWS region decided
- [ ] AWS account ID known

### Infrastructure Decisions
- [ ] VPC strategy clear (new/existing)
- [ ] IaC tool selected (CDK/Terraform)
- [ ] Multi-AZ deployment planned

### Severity if Missing
- No AWS access: **BLOCKER** (cannot deploy)
- Region not decided: **BLOCKER** (must decide before IaC)
- IaC tool not selected: **BLOCKER** (cannot generate code)
- VPC strategy unclear: **GAP** (can decide during IaC)

---

## Database Readiness (if in scope)

### Database Information
- [ ] Database credentials available
- [ ] Connection string format known
- [ ] Database size known
- [ ] Schema complexity understood

### Migration Planning
- [ ] **Backup completed** (CRITICAL)
- [ ] Downtime window scheduled
- [ ] Migration method decided (DMS/Dump-Restore)
- [ ] Validation queries prepared

### DMS Prerequisites (if DMS selected)
- [ ] Engine-specific settings configured (Binlog for MySQL, WAL for PostgreSQL) — see `skills/technical/database-migration.md`
- [ ] DMS VPC Endpoint created (or NAT Gateway available)
- [ ] DMS IAM roles exist (`dms-vpc-role`, `dms-cloudwatch-logs-role`)
- [ ] Security groups allow DMS replication instance → source DB and target DB connectivity
- [ ] Time budget includes DMS setup overhead (~30+ minutes)

### Severity if Missing
- No backup: **BLOCKER** (CRITICAL - cannot proceed)
- Credentials unknown: **BLOCKER** (cannot connect)
- Engine-specific settings not configured (Binlog/WAL): **BLOCKER** (DMS replication will fail)
- DMS VPC Endpoint missing: **GAP** (can create during setup, but adds time)
- Migration method not decided: **GAP** (can decide based on size)
- Downtime not scheduled: **GAP** (can schedule during planning)

---

## Integration Readiness

### External Dependencies
- [ ] All external APIs documented
- [ ] Authentication methods clear
- [ ] API keys/credentials available
- [ ] Rate limits known

### Severity if Missing
- APIs not documented: **GAP** (may discover during testing)
- Credentials missing: **BLOCKER** (if critical integration)
- Rate limits unknown: **ACCEPTABLE** (can monitor)

---

## CI/CD Readiness

### Source Code Repository Access
- [ ] Git repository accessible (clone/pull working)
- [ ] Target branch identified (main/develop/etc.)
- [ ] Branch protection rules understood

### CI/CD Pipeline Prerequisites
- [ ] **GitHub**: Organization Admin permission confirmed (required to install 'AWS Connector for GitHub' app)
- [ ] **CodeCommit**: IAM Git credentials configured
- [ ] **GitLab**: GitLab connector requirements met
- [ ] **Bitbucket**: Bitbucket connector requirements met
- [ ] Pipeline tool selected (CodePipeline / GitHub Actions / etc.)

### Severity if Missing
- GitHub Org Admin permission unavailable: **BLOCKER** (cannot install AWS Connector for GitHub app, CodePipeline integration blocked)
- Git repository inaccessible: **BLOCKER** (cannot set up CI/CD)
- Pipeline tool not selected: **GAP** (can decide during planning)

---

## Team Readiness

### Tools
- [ ] Docker installed and working
- [ ] AWS CLI configured
- [ ] IaC tool installed (CDK/Terraform based on choice)
- [ ] Git installed

### Skills
- [ ] Container knowledge available
- [ ] AWS knowledge available
- [ ] IaC knowledge available

### Severity if Missing
- Docker not installed: **BLOCKER** (cannot build containers)
- AWS CLI not configured: **BLOCKER** (cannot deploy)
- IaC tool not installed: **BLOCKER** (cannot generate infrastructure)
- Skills gap: **GAP** (can learn during execution)

---

## Severity Levels

### BLOCKER (Cannot Proceed)
**Must resolve before starting tasks**:
- No AWS CLI access
- No database backup (if DB migration in scope)
- Container orchestration not decided
- IaC tool not selected
- Critical tools not installed

**Action**: Stop and resolve immediately

### GAP (Can Proceed with Caution)
**Can proceed but may need to pause**:
- Some environment variables unknown
- System dependencies unclear
- Minor configuration unknowns
- Optional features undefined

**Action**: Document gaps, ask when needed during execution

### ACCEPTABLE (Low Risk)
**Can proceed without issue**:
- Optional features undefined
- Nice-to-have information missing
- Can be determined during execution

**Action**: Note and handle during execution

---

## Readiness Check Process

### When to Check
- After requirement analysis
- Before generating task lists
- Before starting task execution

### How to Check
1. Review all analysis documents
2. Go through each checklist item
3. Mark status: ✅ Ready / ⚠️ Gap / 🚫 Blocker
4. Categorize severity
5. Report to user

### If Blockers Found
```markdown
## 🚫 Critical Blockers Identified

Cannot proceed with task execution. Please resolve:

1. [Blocker]: [description]
   - Impact: [what will fail]
   - Action: [what to do]

2. [Blocker]: [description]
   - Impact: [what will fail]
   - Action: [what to do]

Reply "Blockers resolved" when complete.
```

### If Gaps Found
```markdown
## ⚠️ Information Gaps Identified

Can proceed but may need to pause for information:

1. [Gap]: [description]
   - Impact: [when needed]
   - Risk: [Low/Medium/High]

Options:
A) Provide information now (recommended)
B) Proceed and provide when needed

[Answer]:
```

### If Ready
```markdown
## ✅ Readiness Check Passed

All prerequisites met. Ready to proceed with task execution.
```

---

## Integration with Planning Stage

### In planning-stage-guide.md
Before generating task lists:
1. Load readiness-checklist.md
2. Perform readiness check
3. Resolve blockers if found
4. Document gaps if found
5. Only proceed when ready or gaps acceptable
