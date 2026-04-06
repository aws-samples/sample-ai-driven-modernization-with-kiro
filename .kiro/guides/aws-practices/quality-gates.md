# Quality Gates

## Purpose
Define quality standards and approval gates throughout modernization process.

---

## Analysis Stage Quality Gates

### After Application Analysis
- [ ] Technology stack confirmed (not assumed)
- [ ] Dependencies complete (from package files)
- [ ] Configuration requirements documented
- [ ] Integration points identified
- [ ] No "TBD" or "Unknown" in critical fields

### After Database Analysis
- [ ] Database engine and version confirmed
- [ ] Data volume measured (not estimated)
- [ ] Schema complexity assessed
- [ ] Backup strategy documented
- [ ] Connection details verified

### After Infrastructure Analysis
- [ ] Current deployment model documented
- [ ] Resource specifications recorded
- [ ] Network configuration understood
- [ ] Security setup documented
- [ ] Monitoring setup identified

---

## Requirement Analysis Quality Gates

### Requirements Completeness
- [ ] Business goals clearly defined and measurable
- [ ] Technical goals specific with targets
- [ ] Constraints realistic and documented
- [ ] Scope clearly bounded (in/out of scope)
- [ ] NFRs specific and testable
- [ ] Success criteria measurable

### Assessment Quality
- [ ] Five Lenses assessment considered
- [ ] Modernization pattern recommended (Rehost/Replatform/Refactor)
- [ ] Risks identified with mitigations
- [ ] ROI timeframe estimated

---

## Architecture Design Stage Quality Gates

### Architecture Completeness
- [ ] Architecture diagram includes all components, data flow, and integration points
- [ ] Each major decision has inline rationale explaining why
- [ ] Provisioning method selected and documented (IaC or CLI/API)
- [ ] If IaC selected: existing environment, conventions, and state management documented

### Best Practice Compliance
- [ ] Architecture designed following AWS Well-Architected 6 Pillars
- [ ] Modernization strategy aligned with 7Rs
- [ ] Compliance Matrix contains only justified exceptions (not missing items)
- [ ] Each exception has: constraint reason + mitigation plan
- [ ] No unexplained non-compliance (all ⚠️ items confirmed by user)

### Production Cutover (if production environment)
- [ ] Cutover strategy options presented with trade-offs
- [ ] User selected and approved cutover method
- [ ] Green environment setup defined
- [ ] Rollback criteria and procedure documented
- [ ] Pre-cutover checklist defined

### Customer Approval
- [ ] Complete architecture document presented to customer
- [ ] Customer reviewed and provided feedback
- [ ] All feedback incorporated
- [ ] **Explicit approval obtained** (MANDATORY before Stage 4)

---


## Planning Stage Quality Gates

### Task List Quality
- [ ] All phases covered (Foundation → Deployment → Monitoring)
- [ ] Tasks actionable and specific (not vague)
- [ ] Dependencies clearly identified
- [ ] Test/validation steps included in every task
- [ ] Estimated effort reasonable
- [ ] Approval gates defined at phase boundaries
- [ ] Testing/QA Phase included (MANDATORY - unit/integration tests + customer QA)
- [ ] Traffic cutover policy defined (MANDATORY - strategy, rollback criteria, monitoring metrics)

### Dependency and Traceability
- [ ] Every task has `depends_on`, `traces_to`, and `parallel_with` fields
- [ ] All `traces_to` references point to valid requirements or architecture items
- [ ] No circular dependencies in `depends_on` chains
- [ ] Cross-validation completed: all requirements have corresponding tasks
- [ ] Cross-validation completed: all architecture components have corresponding tasks

### Readiness Verification
- [ ] Readiness checklist completed
- [ ] All blockers resolved
- [ ] Gaps documented and acceptable
- [ ] Prerequisites confirmed
- [ ] Tasks align with approved To-Be architecture from Stage 3

### Architecture Consistency (Post-Validation)
- [ ] Validation findings compared against `to-be-architecture.md`
- [ ] If discrepancies found: architecture updated and user re-approval obtained
- [ ] If no discrepancies: architecture validity confirmed
- [ ] Cutover-related IaC changes identified and scoped (if applicable)

---

## Execution Stage Quality Gates

### After Each Task
- [ ] Task executed successfully (no errors)
- [ ] Expected output generated
- [ ] Verification steps passed
- [ ] Checkbox updated in task file
- [ ] No pending issues

### After Each Phase

**Phase 1: Foundation**
- [ ] VPC and networking created
- [ ] Security groups configured
- [ ] ECR repository created
- [ ] Basic monitoring setup
- [ ] All resources tagged

**Phase 2: Application Containerization**
- [ ] Dockerfile created with multi-stage build
- [ ] Container built successfully
- [ ] **Local test passed** (MANDATORY)
- [ ] **Security scan passed** (MANDATORY - no HIGH/CRITICAL)
- [ ] Image pushed to ECR

**Phase 3: Infrastructure Deployment**
- [ ] IaC code generated
- [ ] Security scan passed (cdk-nag/tfsec)
- [ ] Deployed to dev/staging
- [ ] Resources verified
- [ ] Connectivity tested

**Phase 4: Application Deployment**
- [ ] ECS/EKS cluster created
- [ ] Application deployed
- [ ] Health checks passing
- [ ] Auto-scaling configured
- [ ] Load balancer working

**Phase 5: Testing**
- [ ] Integration tests passed
- [ ] Performance tests passed
- [ ] Security tests passed
- [ ] All acceptance criteria met

**Phase 6: Production Deployment**
- [ ] Deployed to production
- [ ] Smoke tests passed
- [ ] Monitoring validated
- [ ] Rollback tested

---

## Mandatory Quality Standards

### Container Quality (MANDATORY)
- ✅ Multi-stage build used
- ✅ Multi-architecture build (amd64 + arm64)
- ✅ Non-root user configured
- ✅ Local test passed before ECR push
- ✅ Security scan passed (0 HIGH/CRITICAL vulnerabilities)
- ❌ NEVER push untested images
- ❌ NEVER skip security scans

### Infrastructure Quality (MANDATORY)
- ✅ Multi-AZ deployment (minimum 2 AZs)
- ✅ Encryption enabled (at-rest and in-transit)
- ✅ Least privilege IAM roles
- ✅ Security groups restrictive (no 0.0.0.0/0 except ALB)
- ✅ All resources tagged
- ✅ Deployed to dev/staging before production

### Database Quality (MANDATORY - if applicable)
- ✅ **Backup completed before migration** (CRITICAL)
- ✅ Data validation passed (record counts match)
- ✅ Rollback tested
- ✅ Connection verified
- ❌ NEVER migrate without backup

### Testing Quality (MANDATORY)
- ✅ Unit tests: 80%+ coverage (if applicable)
- ✅ Integration tests passed
- ✅ Performance targets met
- ✅ Security scan passed
- ✅ Smoke tests passed after deployment

---

## Approval Gates

### When to Request Approval

**MUST request approval**:
- After completing each major phase
- Before deploying to production
- Before database migration cutover
- Before making irreversible changes

**Can proceed without approval**:
- After generating code (user reviews later)
- After building containers (tested locally)
- After deploying to dev/staging
- After running tests

### Approval Message Format
```markdown
# 🔍 Checkpoint: [Phase Name]

## Completed Work
[Summary]

## Results
[Key outcomes and metrics]

## Next Steps
[What will happen next]

## Risk Assessment
- Risk Level: [Low/Medium/High]
- Reversible: [Yes/No]
- Rollback Plan: [Available/Not needed]

## Please Review and Approve
- [ ] Quality standards met
- [ ] Results acceptable
- [ ] Ready to proceed

Type 'APPROVED' to continue.
```

---

## Failure Handling

### When Quality Gate Fails
1. Stop execution
2. Analyze failure
3. Attempt automatic fix
4. Retry validation
5. If cannot fix: Escalate to user

### Escalation Template
```markdown
## ❌ Quality Gate Failed

**Gate**: [which gate]
**Issue**: [what failed]
**Standard**: [what was expected]
**Actual**: [what was found]

**Options**:
A) Fix and retry
B) Accept risk and proceed (document reason)
C) Rollback and try different approach

[Answer]:
```

---

## Success Criteria

### Quality gates are effective when
- ✅ Consistent quality across all phases
- ✅ Issues caught early
- ✅ No untested code in production
- ✅ All security standards met
- ✅ Clear go/no-go decisions
