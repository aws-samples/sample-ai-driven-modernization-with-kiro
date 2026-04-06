# Execution Stage Guide

## Purpose
Guide AI through executing tasks from task list files.

---

## Execution Rules

### Sequential Execution
1. Open task files in numerical order (1, 2, 3...)
2. Execute tasks within each file sequentially
3. Update checkbox [ ] → [x] when complete
4. Verify before moving to next task
5. Move to next file only when current file complete

### Task Execution Pattern
```
For each task:
1. Read task description and subtasks
2. Execute each subtask using appropriate tools
3. Verify result after each subtask
4. Update checkbox [ ] → [x] when subtask complete
   ⚠️ MANDATORY: Do NOT proceed to next task without updating checkbox
5. If all subtasks complete: Mark main task complete
6. If error: Follow error handling procedure
7. Move to next task
```

### Checkbox Update Rule (MANDATORY)
- **Every task completion MUST be immediately followed by checkbox update** `[ ] → [x]`
- The sequence is: Execute → Verify → **Update checkbox** → Next task
- **Do NOT batch checkbox updates** — update each task individually right after completion
- If a task is partially complete, leave it as `[ ]` until fully done

---

## Tool Usage

### Available Tools
- `execute_bash`: Shell commands (docker, git, npm, etc.)
- `use_aws`: AWS CLI operations
- `fs_write`: Create/modify files (Dockerfile, IaC code, configs)
- `fs_read`: Read files for validation

### Examples

**Docker operations**:
```bash
execute_bash: docker build -t myapp:latest .
execute_bash: docker run -d -p 8080:8080 myapp:latest
execute_bash: docker logs [container-id]
```

**AWS operations**:
```
use_aws: ecr create-repository --repository-name myapp
use_aws: ecs create-cluster --cluster-name myapp-cluster
```

**File operations**:
```
fs_write: Create Dockerfile
fs_write: Create infrastructure/lib/network-stack.ts
fs_read: Read package.json to verify dependencies
```

---

## Validation After Each Task

### Checklist
- [ ] Task executed successfully (no errors)
- [ ] Expected output generated
- [ ] Verification steps passed
- [ ] Checkbox updated in task file
- [ ] Ready for next task

### Verification Methods
- Check command exit codes
- Verify files created
- Test endpoints (curl, health checks)
- Review logs for errors
- Validate resource creation (AWS CLI)

---

## Error Handling

### When Error Occurs

**Step 1: Detect**
- Capture error message
- Note which task failed
- Record context (what was being done)

**Step 2: Analyze**
- Read error message carefully
- Identify error type (build, runtime, deployment, permission)
- Determine likely cause

**Step 3: Attempt Fix**
- Try automatic remediation
- Common fixes:
  - Missing dependency → Add to Dockerfile
  - Permission error → Check IAM/security groups
  - Connection error → Verify endpoints/credentials
  - Build error → Check syntax, versions

**Step 4: Retry**
- Execute task again
- Verify if fixed

**Step 5: Escalate (if cannot fix after 2 attempts)**
```markdown
## ⚠️ Task Failed - Need Your Input

**Task**: [task name]
**Error**: [error message]

**What I tried**:
1. [Attempt 1] - Failed
2. [Attempt 2] - Failed

**Analysis**: [likely cause]

**Options**:
A) [Suggested fix 1]
B) [Suggested fix 2]
C) Skip this task
D) Provide more information

[Answer]:
```

**Step 6: Resume**
- Based on user decision
- Continue or rollback

---

## Progress Tracking

### Update Task Files
- Mark completed: [ ] → [x]
- Keep incomplete: [ ]
- Add notes if needed: [x] (with modifications)

### Report Progress
After each major task:
```markdown
✅ Task completed: [task name]
- Result: [what was created/configured]
- Validation: [test results]
- Next: [next task name]
```

---

## Working Directory

### Directory Structure
```
<workspace-root>/
├── source-code/
│   ├── backend/          # Work here for backend tasks
│   └── frontend/         # Work here for frontend tasks
├── infrastructure/       # Work here for IaC tasks
└── outputs/
    └── tasks/            # Task files here
```

### Navigation
- Start from workspace root
- Navigate to appropriate directory for each task
- Return to root after task completion

---

## Validation Standards

### Before Moving to Next Task
- [ ] Current task fully complete
- [ ] No errors in execution
- [ ] Verification passed
- [ ] Checkbox updated
- [ ] Output documented

### Before Moving to Next File
- [ ] All tasks in current file complete
- [ ] All validations passed
- [ ] No pending issues
- [ ] Ready for next phase

---

## Common Task Types

### Infrastructure Tasks
- Create VPC, subnets, security groups
- Use IaC (CDK/Terraform)
- Deploy and verify resources
- Test connectivity

### Container Tasks
- Create Dockerfile
- Build image locally
- Test container
- Scan for vulnerabilities
- Push to ECR

### Deployment Tasks
- Create ECS/EKS cluster
- Define task/pod specs
- Deploy application
- Configure auto-scaling
- Setup load balancer

### CI/CD Tasks
- Create pipeline
- Configure build stage
- Configure deploy stage
- Test pipeline execution

### Monitoring Tasks
- Create CloudWatch dashboards
- Configure alarms
- Setup log aggregation
- Test alerting
