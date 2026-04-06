# Planning Stage Guide

## Purpose
Guide AI through creating modernization plan and detailed task lists.

---

## Objectives
- Generate phase-by-phase task lists based on approved To-Be architecture
- Ensure safety measures (backup/rollback)
- Include validation steps
- Align tasks with architecture decisions from Stage 3

---

## Planning Process

### Step 1: Review All Analysis
### Step 1: Review All Analysis and Architecture
- Load as-is-analysis-application.md
- Load as-is-analysis-database.md (if applicable)
- Load as-is-analysis-infrastructure.md
- Load requirements-analysis.md
- **Load approved to-be-architecture.md from Stage 3** (MANDATORY)
- Verify Stage 3 approval was obtained

### Step 1: Review Approved Architecture
- Load approved `outputs/architecture/to-be-architecture.md` from Stage 3
- Verify customer approval was obtained
- Review architecture decisions, provisioning method (IaC/CLI), and cutover strategy
- Load requirements-analysis.md for cross-reference

### Step 2: Generate Task Lists

**Create separate task files for each phase**:

1. **Foundation Infrastructure**: VPC, Subnets, Security Groups, IAM Roles, ECR
2. **Application Containerization**: Dockerfile creation
3. **Container Image Testing**: Local build, test, security scan
4. **ECR and Image Push**: ECR push, tagging
5. **ECS/EKS Cluster Setup**: Cluster creation
6. **Green Environment Deployment**: Green ALB, Service, application deployment
7. **CI/CD Pipeline Configuration**: CodePipeline or GitOps
8. **Monitoring Setup**: CloudWatch, alarms, dashboards
9. **Testing and QA** (MANDATORY): Unit/integration test execution + customer QA
10. **Traffic Cutover Policy** (MANDATORY): Cutover strategy, rollback criteria, monitoring metrics

**Phase 9, 10 Mandatory Rules**:
- Phase 9 (Testing and QA): Unit tests and integration tests must be executed. Provide endpoints and test checklists so the customer can perform QA directly on the Green environment. Task lists without this Phase cannot be generated.
- Phase 10 (Traffic Cutover Policy): Cutover strategy (Canary/Blue-Green/Rolling, etc.), rollback criteria (error rate, response time, etc.), and monitoring metrics must be defined. The execution method (automated scripts vs manual) is determined by user requirements. Task lists without this Phase cannot be generated.

---

## Task File Format

### Template
```markdown
# [N]. [Phase Name] Tasks

## Objective
[Goal of this phase]

## Prerequisites
- [Previous phase completed]
- [Required information/tools]

## Task List

### 1. [Task Name]
- [ ] [Subtask 1]
- [ ] [Subtask 2]
- [ ] Test and verify

### 2. [Task Name]
- [ ] [Subtask]
- [ ] Test and verify

## Validation Criteria
- [ ] [Validation item 1]
- [ ] [Validation item 2]

## Next Phase
[Next file name]
```

---

## Task Design Principles

### Task Scope Separation (MANDATORY)

**ECS/EKS task files and Database migration task files MUST be completely separate.**

Rules:
- ECS/EKS tasks (`outputs/tasks/ecs/` or `outputs/tasks/eks/`) must NOT include database migration, schema change, or DMS-related tasks
- Database migration tasks must go to `outputs/tasks/database/` as separate files
- The only DB-related content allowed in ECS/EKS tasks is **connection configuration** (e.g., setting DB endpoint in environment variables or Secrets Manager)
- If the modernization plan includes both container and DB modernization, generate two independent sets of task files
- Cross-validation must verify this separation: flag any DB migration task found in ECS/EKS files as an error

**Allowed in ECS/EKS tasks**: DB connection string configuration, Secrets Manager reference for DB credentials
**NOT allowed in ECS/EKS tasks**: DB engine migration, DMS setup, schema migration, data validation, DB backup/restore

### Safety First (MANDATORY)
**Every task list MUST include**:
- Backup tasks before destructive operations
- Rollback procedures documented
- Validation after each change
- Checkpoint creation

**Specifically**:
- Database backup before migration (CRITICAL)
- Container image tagging (for rollback)
- Infrastructure state backup (IaC)

### Granularity
- Each task: 15-30 minutes
- Specific and actionable
- Include test/validation steps

### Sequencing
- Infrastructure before application
- Build before deploy
- Test after each step
- Dependencies clearly ordered

### Dependency Management (MANDATORY)
Each task MUST include dependency metadata:
- **`depends_on`**: List of prerequisite tasks that must complete before this task can start
- **`traces_to`**: Reference to the requirement or analysis item this task addresses
- **`parallel_with`**: Tasks that can run concurrently with this task (if any)

Rules:
- No task may start without its `depends_on` tasks completed
- Every requirement from the requirements analysis must have at least one task with `traces_to` pointing to it
- Identify parallelizable tasks to optimize execution time
- No circular dependencies allowed

---

### Cross-Validation After Task Generation (MANDATORY)

After generating all task files, AI must perform cross-validation:

1. **Requirements coverage check**: For each item in `requirements-analysis.md`, verify at least one task addresses it
   - If missing: Generate warning "Requirement [X] has no corresponding task"
2. **Architecture alignment check**: For each component in `to-be-architecture.md`, verify tasks exist to create/configure it
   - If missing: Generate warning "Architecture component [X] has no corresponding task"
3. **Dependency consistency check**: Verify no circular dependencies exist and all `depends_on` references point to valid tasks
4. **Architecture feedback check**: If cross-validation reveals gaps between tasks and architecture:
   - Identify components in `to-be-architecture.md` that need updates (e.g., missing resources, changed scope)
   - Propose architecture updates to user with highlighted changes
   - Require user re-approval of architecture changes before finalizing tasks
5. **Cutover IaC impact check** (if production cutover planned):
   - List all IaC Stacks/modules that need modification during cutover tasks
   - If 3+ Stacks require changes → ⚠️ warn and suggest Stack re-separation
   - Verify cutover tasks explicitly document which IaC files/Stacks are affected
6. **Present validation results to user** before finalizing task lists

---

## Phase Details

### Phase 1: Foundation Infrastructure
**Scope**: Network and security foundation
- VPC, Subnets (Public/Private, Multi-AZ)
- Internet Gateway, NAT Gateway
- Route Tables
- Security Groups (ALB, ECS/EKS, RDS)
- IAM Roles (Task Role, Execution Role)
- ECR Repository
- CloudWatch Log Groups

**Not included**: ECS/EKS cluster, ALB, Application

---

### Phase 5: ECS/EKS Cluster Setup
**Scope**: Container orchestration platform only
- ECS Cluster creation (empty cluster)
- Or EKS Cluster creation
- Basic configuration (logging, monitoring)
- Cluster-level IAM setup

**Not included**: Application deployment, ALB

---

### Phase 6: Green Environment Deployment
**Scope**: Deploy application to Green environment with separate endpoint
- **Green ALB creation** (separate DNS endpoint)
- Green Target Group
- ECS Service creation (runs application)
- Or K8s Deployment + Service + Ingress
- Connect to database
- Application actually running
- **Output Green endpoint URL for customer QA**

**Key point**: 
- Green environment is production-ready
- Has separate endpoint (not connected to production traffic yet)
- Customer can test via Green endpoint
- Traffic is 0% (standby for cutover)

---

## Output Format

### Plan File
`outputs/plan/modernization-plan-[ecs|eks|database].md`

**Required sections**:
1. Approved Architecture Reference (link to Stage 3 output)
2. Migration Strategy (pattern, approach)
3. Phase Overview (phases with descriptions)
4. Success Criteria (measurable)

### Task Files
`outputs/tasks/[ecs|eks|database]/[1-8]. [Phase Name] Tasks.md`

**File naming**:
- Use numbers for ordering
- Use descriptive names
- Include "Tasks.md" suffix

---

## Validation

### Plan Validation
- [ ] All phases covered (1-8)
- [ ] Architecture diagram clear and complete
- [ ] Strategy aligned with requirements
- [ ] Success criteria measurable
- [ ] Green environment with separate endpoint included

### Task List Validation
- [ ] All tasks actionable and specific
- [ ] Dependencies identified and ordered
- [ ] Test/validation steps included in every task
- [ ] Backup/rollback tasks included where needed
- [ ] Phase 6 includes Green ALB with separate endpoint
- [ ] Estimated effort reasonable
- [ ] Ready for execution

---

## Integration with Other Guides

### Reference During Planning (Skills — auto-loaded on demand)
- `skills/aws-practices/modernization-strategy.md` - Pattern selection
- `skills/aws-practices/container-orchestration-selection.md` - ECS vs EKS
- `skills/technical/container-best-practices.md` - Container tasks
- `skills/technical/infrastructure-as-code.md` - IaC tasks
- `skills/technical/database-migration.md` - DB tasks
- `guides/common/output-templates.md` - Output formats

### Ensure Alignment
- Task lists match selected pattern
- Technology choices consistent
- Safety measures included
- Quality standards applied
