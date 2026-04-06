# Output Templates

## Purpose
Standard templates for all output documents to ensure consistency.

## Language Adaptation Rule
**MANDATORY**: All section titles, labels, and content in output documents must be generated in the user's detected language (see `steering/workflow-orchestration.md` > Language Detection and Adaptation).

- The templates below are written in English as a structural reference
- When generating actual output, translate all section titles, labels, field names, and descriptions to the user's language
- Technical terms (e.g., AWS service names, CLI commands, configuration keys) remain in English
- Example: "## Application Overview" → translated to the user's language (e.g., Korean, Spanish)

---

## Analysis Output Template

### Application Analysis
```markdown
# As-Is Analysis - Application

## Application Overview
- **Name**: [application name]
- **Purpose**: [business purpose]
- **Type**: [Web/API/Batch/etc.]
- **Users**: [who uses it]

## Technology Stack
- **Language**: [Java/Node.js/Python/etc.]
- **Framework**: [Spring Boot/Express/Django/etc.]
- **Version**: [specific version]
- **Runtime**: [JVM/Node/Python version]

## Code Structure
```
source-code/[repo-name]/
├── src/
├── tests/
├── config/
└── [package file]
```

## Dependencies
- [package-1]: [version] - [purpose]
- [package-2]: [version] - [purpose]

## Configuration
- **Port**: [port number]
- **Health Check**: [endpoint path or "Not implemented"]
- **Environment Variables**:
  - `VAR_NAME`: [purpose]
  - `VAR_NAME`: [purpose]

## Current Deployment
- **Method**: [How deployed - VM/Container/Manual]
- **Location**: [On-premises/AWS/Other]
- **Process**: [Deployment steps]

## Integration Points
- **Inbound**: [Systems calling this app]
- **Outbound**: [Systems this app calls]

## Operational Characteristics
- **Deployment**: [tool, frequency, zero-downtime capability]
- **Long-running Tasks**: [batch jobs, workers, schedulers — duration, shutdown behavior]
- **Graceful Shutdown**: [SIGTERM handling, drain timeout]
- **Process Structure**: [single/multi-process, sidecar patterns]

## Issues Identified
- [Issue 1]: [description and impact]
- [Issue 2]: [description and impact]
```

### Database Analysis
```markdown
# As-Is Analysis - Database

## Database Overview
- **Engine**: [MySQL/PostgreSQL/Oracle/etc.]
- **Version**: [specific version]
- **Size**: [GB]
- **Location**: [host:port]

## Schema Structure
- **Tables**: [count]
- **Stored Procedures**: [count]
- **Triggers**: [count]
- **Views**: [count]

## Data Volume
- **Current Size**: [GB]
- **Growth Rate**: [GB/month]
- **Largest Tables**: [list with sizes]

## Complexity Assessment
- **Relationships**: [Simple/Moderate/Complex]
- **Stored Procedures**: [None/Few/Many]
- **Data Quality**: [Good/Issues identified]

## Current Backup
- **Method**: [mysqldump/snapshot/etc.]
- **Frequency**: [daily/weekly]
- **Retention**: [days]
- **Last Backup**: [date]

## Connection Details
- **Host**: [hostname or IP]
- **Port**: [port]
- **Database Name**: [name]
- **Credentials**: [location - Secrets Manager/Parameter Store/File]

## Migration Considerations
- **Downtime Tolerance**: [hours]
- **Recommended Method**: [DMS/Dump-Restore]
- **Estimated Duration**: [hours]
```

### Infrastructure Analysis
```markdown
# As-Is Analysis - Infrastructure

## Infrastructure Overview
- **Deployment Model**: [On-premises/AWS/Hybrid]
- **Environment**: [Production/Staging/Dev]

## Compute Resources
- **Type**: [Physical/VM/Container]
- **Count**: [number]
- **Specs**: [CPU, Memory, Storage]
- **OS**: [operating system]

## Network Configuration
- **VPC**: [ID or "Not applicable"]
- **Subnets**: [public/private configuration]
- **Security Groups**: [rules summary]
- **Load Balancer**: [type and configuration]

## Storage Resources
- **Type**: [EBS/S3/EFS/Local]
- **Size**: [GB]
- **Purpose**: [application data/logs/uploads]

## Security Setup
- **Firewall**: [configuration]
- **IAM Roles**: [if AWS]
- **Encryption**: [at-rest/in-transit]
- **Access Control**: [method]

## Monitoring
- **Tools**: [CloudWatch/Prometheus/etc.]
- **Metrics**: [what's monitored]
- **Logs**: [where logs go]
- **Alerts**: [alerting setup]

## Deployment Process
- **Method**: [Manual/Automated]
- **Frequency**: [how often]
- **Rollback**: [capability]

## Environment Constraints
- **Security Policies**: [firewalls, proxies, certificate management, compliance]
- **Network Restrictions**: [NAT Gateway, VPN, Direct Connect, IDC connectivity]
- **Reusable Resources**: [existing clusters, ALBs, VPCs that can be reused]

## Server Internals
- **OS**: [type and version, e.g., Amazon Linux 2023, Ubuntu 22.04]
- **Compute Specs**: [CPU cores, memory, storage type/size, network I/O]
- **Running Processes**: [list of processes beyond main app — agents, sidecars, cron jobs]
- **Configuration Files**: [key config files and locations — nginx.conf, supervisord.conf, etc.]
- **Installed Packages**: [system-level dependencies and libraries]
- **Log Collection**: [method — file-based, CloudWatch Agent, Fluentd, etc.]
```

---

## Architecture Design Output Template

### To-Be Architecture Document (AI Working Document)
**File**: `outputs/architecture/to-be-architecture.md`

```markdown
# To-Be Architecture

## Architecture Overview
[High-level description of target architecture and key decisions]

## Constraints
[Constraints confirmed with customer in Step 2]
- [Constraint 1]: [description]
- [Constraint 2]: [description]

## Provisioning Method
- **Method**: [IaC (CDK/Terraform) | CLI/API]
- **Rationale**: [why this method was selected]
- **Existing Environment**: [existing IaC setup details, if applicable]

## Architecture Design

### [Component 1]
[Description and configuration]
→ Rationale: [why this decision follows best practice]

### [Component 2]
[Description and configuration]
→ Rationale: [why this decision follows best practice]

⚠️ [Exception item]
→ Best Practice: [what should be done]
→ Constraint: [why it cannot be followed]
→ Mitigation: [alternative approach]

## Architecture Diagram
```mermaid
[Mermaid diagram]
```

## Compliance Matrix (Exceptions Only)

| Item | Best Practice | Current Decision | Constraint | Mitigation |
|------|--------------|-----------------|------------|------------|
| [item] | [best practice] | [what we do] | [why] | [alternative] |

## Cutover Strategy
[Only if production environment]
- **Method**: [selected cutover method]
- **Green Environment**: [setup details]
- **QA Procedure**: [test plan on Green endpoint]
- **Rollback Criteria**: [conditions for rollback]
- **Pre-Cutover Checklist**: [items to verify]

## Approval
- **Status**: [Pending / Approved]
- **Date**: [approval date]
- **Notes**: [customer feedback]
```

### To-Be Architecture HTML (Customer Review)
**File**: `outputs/architecture/to-be-architecture.html`

Generate an HTML file that renders the same Mermaid diagram with custom theme:
- Mermaid CDN (`https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js`)
- Custom theme: component-type colors (Compute=orange, Database=blue, Network=green, Security=red)
- Page layout: title, diagram, legend, notes section
- Responsive design for browser viewing

---


## Planning Output Template

### Modernization Plan
```markdown
# Modernization Plan - [ECS/EKS/Database]

## Target Architecture

### Overview
[High-level description of target state]

### Key Components
- [Component 1]: [description]
- [Component 2]: [description]

### Architecture Diagram
```mermaid
[Mermaid diagram]
```

## Migration Strategy
- **Pattern**: [Rehost/Replatform/Refactor]
- **Approach**: [Phased/Big-bang]
- **Timeline**: [estimated duration]

## Phase Overview
1. **Phase 1**: [name] - [purpose]
2. **Phase 2**: [name] - [purpose]
...

## Success Criteria
- [ ] [Criterion 1]
- [ ] [Criterion 2]

## Risks and Mitigations
- **Risk**: [description] - **Mitigation**: [plan]
```

---

## Task List Template

```markdown
# [Phase Number]. [Phase Name] Tasks

## Objective
[Goal of this phase]

## Prerequisites
- [Previous phase completed]
- [Required tools/permissions]

## Task List

### 1. [Task Name]
**Goal**: [Goal of this task]
**depends_on**: [Prerequisite task IDs, e.g., "Phase 1 > Task 2" or "None"]
**traces_to**: [Requirement or architecture item this addresses]
**parallel_with**: [Tasks that can run concurrently, or "None"]

- [ ] [Subtask 1]
- [ ] [Subtask 2]
- [ ] [Subtask 3]
- [ ] Test: [Test method]
- [ ] Verify: [Verification criteria]

**Estimated Time**: [minutes]

### 2. [Task Name]
**Goal**: [Goal of this task]
**depends_on**: [Prerequisite task IDs]
**traces_to**: [Requirement or architecture item]
**parallel_with**: [Concurrent tasks or "None"]

- [ ] [Subtask]
- [ ] Test: [Test method]
- [ ] Verify: [Verification criteria]

**Estimated Time**: [minutes]

## Validation Criteria
- [ ] [Validation item 1]
- [ ] [Validation item 2]

## Completion Criteria
- [ ] All task checkboxes completed
- [ ] All validations passed
- [ ] Ready for next phase

## Next Phase
[Next task file name]
```

---

## Best Practices

### Task Writing
- Use action verbs (Create, Configure, Deploy, Test)
- Be specific (not "Setup database" but "Create RDS MySQL instance with Multi-AZ")
- Include validation (every task should have test/verify step)
- Consider rollback (note how to undo if needed)

### Dependency Management
- List prerequisites at file start
- Order tasks by dependency
- Note blocking tasks clearly
- Allow parallel tasks where possible

### Documentation
- Document decisions made
- Note deviations from plan
- Record issues encountered
- Update task files with actual results
