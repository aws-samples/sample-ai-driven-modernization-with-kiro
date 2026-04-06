# Analysis Stage Guide

## Purpose
Guide AI through as-is analysis of application, database, and infrastructure.

---

## Application Analysis

### Objectives
- Identify technology stack and versions
- Understand code structure and dependencies
- Document configuration requirements
- Identify integration points
- Assess operational characteristics and constraints

### Analysis Checklist
- [ ] **Target environment identified** (development/staging/production) — if production, flag "safe cutover strategy required"
- [ ] Programming language and framework identified
- [ ] Runtime version confirmed
- [ ] Dependencies analyzed (package.json, pom.xml, requirements.txt)
- [ ] Application port identified
- [ ] Health check endpoint identified (if exists)
- [ ] Environment variables documented
- [ ] External integrations identified
- [ ] Current deployment method understood
- [ ] **Deployment details**: deployment tool, frequency, zero-downtime capability
- [ ] **Long-running tasks**: batch jobs, workers, schedulers identified
- [ ] **Graceful shutdown**: SIGTERM handling, drain timeout behavior
- [ ] **Process structure**: single vs multi-process, sidecar patterns
- [ ] **Environment separation**: env vars, config files, profiles
- [ ] **External integration details**: auth methods, network paths, timeouts

### Output Format
**File**: `outputs/analysis/as-is-analysis-application.md`

**Required sections**:
1. Application Overview (name, purpose, type)
2. Technology Stack (language, framework, version)
3. Code Structure (directories, key components)
4. Dependencies (from package files)
5. Configuration (env vars, config files, ports)
6. Current Deployment (method, location)
7. Integration Points (inbound, outbound)
8. **Operational Characteristics** (deployment, long-running tasks, shutdown behavior)
9. Issues Identified (problems, limitations)

---

## Database Analysis

### Objectives
- Identify database engine and version
- Understand data volume and complexity
- Document schema structure
- Assess migration requirements

### Analysis Checklist
- [ ] Database engine and version identified
- [ ] Database size determined
- [ ] Number of tables counted
- [ ] Schema complexity assessed
- [ ] Stored procedures/triggers identified
- [ ] Data quality issues noted
- [ ] Backup strategy understood
- [ ] Connection details documented

### Output Format
**File**: `outputs/analysis/as-is-analysis-database.md`

**Required sections**:
1. Database Overview (engine, version, size)
2. Schema Structure (tables, relationships)
3. Data Volume (size, growth rate)
4. Complexity (stored procedures, triggers, jobs)
5. Data Quality (issues, cleanup needed)
6. Current Backup (strategy, frequency)
7. Connection Details (host, port, credentials location)
8. Migration Considerations (downtime, method)

---

## Infrastructure Analysis

### Objectives
- Document current infrastructure setup
- Identify compute, network, storage resources
- Understand deployment architecture
- Assess cloud readiness
- Identify security/network constraints and reusable resources

### Analysis Checklist
- [ ] Compute resources identified (servers, VMs, containers)
- [ ] Network configuration documented
- [ ] Storage resources identified
- [ ] Load balancing setup understood
- [ ] Security configuration documented
- [ ] Monitoring setup identified
- [ ] Deployment process documented
- [ ] AWS resources identified (if applicable)
- [ ] **Security program/policy constraints**: firewalls, proxies, certificate management, compliance requirements
- [ ] **Network constraints**: NAT Gateway, VPN, Direct Connect, IDC connectivity
- [ ] **Reusable resources identified**: existing ECS/EKS clusters, ALBs, VPCs, security groups
- [ ] **Server internals identified**:
  - OS type and version (e.g., Amazon Linux 2, Ubuntu 22.04)
  - Compute specs (CPU cores, memory, storage type/size, network I/O)
  - Running processes beyond the main application (agents, sidecars, cron jobs, log collectors)
  - Configuration files and locations (nginx.conf, supervisord.conf, logrotate, etc.)
  - System-level installed packages/libraries (rpm -qa / dpkg -l)
  - Log collection method (file-based, CloudWatch Agent, Fluentd, etc.)

### AWS CLI Auto-Discovery Commands (MANDATORY)
Before asking the user about infrastructure, run these commands to auto-discover:
```bash
# EC2 instances — count, type, state, tags
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name,PrivateIpAddress,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' --output table

# ALB/NLB — load balancer existence and configuration
aws elbv2 describe-load-balancers --query 'LoadBalancers[].[LoadBalancerName,Type,Scheme,DNSName,VpcId]' --output table

# Target Groups — what's behind the load balancer
aws elbv2 describe-target-groups --query 'TargetGroups[].[TargetGroupName,Port,Protocol,TargetType]' --output table

# Security Groups — inbound/outbound rules
aws ec2 describe-security-groups --query 'SecurityGroups[].[GroupId,GroupName,Description]' --output table

# VPC and Subnets
aws ec2 describe-vpcs --query 'Vpcs[].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' --output table
aws ec2 describe-subnets --query 'Subnets[].[SubnetId,VpcId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' --output table

# RDS/Aurora instances
aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,Engine,EngineVersion,DBInstanceClass,DBInstanceStatus,Endpoint.Address]' --output table

# ECS clusters (check for existing clusters to reuse)
aws ecs list-clusters

# ECR repositories (check for existing repos)
aws ecr describe-repositories --query 'repositories[].[repositoryName,repositoryUri]' --output table

# Route53 hosted zones (if DNS management needed)
aws route53 list-hosted-zones --query 'HostedZones[].[Name,Id]' --output table
```

### Server Internals Discovery (MANDATORY)
AI must attempt to access servers via SSM first, then SSH. Do NOT skip this step.

**Step 1: Check SSM availability**
```bash
# Check if instances have SSM agent running
aws ssm describe-instance-information --query 'InstanceInformationList[].[InstanceId,PingStatus,PlatformType,PlatformName]' --output table
```

**Step 2a: If SSM available — run commands via SSM**
```bash
# Run discovery commands via SSM
aws ssm send-command \
  --instance-ids "[instance-id]" \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["uname -a","cat /etc/os-release","nproc","free -h","df -h","ps aux --sort=-%mem | head -20","systemctl list-units --type=service --state=running","crontab -l 2>/dev/null","netstat -tlnp 2>/dev/null || ss -tlnp","env | sort","ls -la /etc/nginx/ /etc/supervisor/ /etc/logrotate.d/ 2>/dev/null"]'

# Retrieve command output
aws ssm get-command-invocation --command-id "[command-id]" --instance-id "[instance-id]"
```

**Step 2b: If SSM unavailable — request SSH access from user**
```
SSM Agent is not installed or not responding.
SSH access information is required to inspect the server internals:
- SSH key path or access method
- Bastion host (if available)
```

**Step 2c: If neither SSM nor SSH available — document as gap**
```
⚠️ Server internal inspection unavailable: Neither SSM nor SSH accessible
→ Server internal information (processes, configs, packages) must be provided by the user
→ Add server internal information questions to plan.md
```

**Step 3: When access is available, gather server internals**
```bash
# OS info
uname -a && cat /etc/os-release
# Compute specs
nproc && free -h && df -h && lsblk
# Running processes
ps aux --sort=-%mem | head -20
systemctl list-units --type=service --state=running
# Listening ports
netstat -tlnp 2>/dev/null || ss -tlnp
# Environment variables
env | sort
# Cron jobs
crontab -l 2>/dev/null; ls /etc/cron.d/
# Config files
ls -la /etc/nginx/ /etc/supervisor/ /etc/logrotate.d/ 2>/dev/null
# Installed packages
rpm -qa 2>/dev/null || dpkg -l 2>/dev/null
```

### Output Format
**File**: `outputs/analysis/as-is-analysis-infrastructure.md`

**Required sections**:
1. Infrastructure Overview (deployment model)
2. Compute Resources (servers, specs, count)
3. Network Configuration (VPC, subnets, security groups)
4. Storage Resources (volumes, buckets, file systems)
5. Load Balancing (type, configuration)
6. Security Setup (firewalls, IAM, encryption)
7. Monitoring (tools, metrics, logs)
8. Deployment Process (how deployments happen)
9. **Server Internals** (OS, running processes, config files, installed packages, log collection)
10. **Environment Constraints** (security policies, network restrictions, reusable resources)

---

## Common Patterns

### Information Gathering Priority (MANDATORY)

Before asking the user any question, AI must attempt to gather information in this order:

```
Priority 1 — Source Code Direct Analysis:
  - Language, framework, version → Read package.json / pom.xml / requirements.txt / go.mod
  - Dependencies → Parse dependency files
  - Application port → Read config files, Dockerfile, application.yml, .env
  - Environment variables → Read .env, config files, source code references
  - Integration points → Search for HTTP client calls, SDK usage, queue connections
  - Health check endpoints → Search for /health, /ready, /ping routes

Priority 2 — AWS CLI Auto-Discovery:
  - EC2 instances → aws ec2 describe-instances
  - Load balancers → aws elbv2 describe-load-balancers
  - Security groups → aws ec2 describe-security-groups
  - VPC/Subnets → aws ec2 describe-vpcs, aws ec2 describe-subnets
  - RDS/Aurora → aws rds describe-db-instances
  - Target groups → aws elbv2 describe-target-groups
  - See "AWS CLI Auto-Discovery Commands" section below for full list

Priority 3 — SSH/SSM Server Internals:
  - Running processes, ports, config files, installed packages
  - See "Server Internals Discovery" section below

Priority 4 — User Questions (LAST RESORT):
  - ONLY for information that cannot be discovered by the above methods
  - Business-specific context (peak hours, batch schedules, SLA)
  - Regulatory/compliance requirements
  - Workload-specific operational conditions
  - Future plans, strategic decisions
  - External integration contract terms, authentication agreements
  - Organizational constraints (team skills, budget, timeline)
```

**CRITICAL**: Do NOT ask the user questions that can be answered by reading source code, running AWS CLI commands, or accessing servers via SSH/SSM. Gather first, ask only what remains unknown.

### Information Gathering

**Step 0: Detect document status and collect initial settings (MANDATORY)**

Before starting analysis, collect environment settings and check input documents:

```
1. Auto-detect available settings:
   - Run: aws sts get-caller-identity → capture Account ID
   - Run: aws configure get region → capture Region
   - Run: git remote -v → capture repository URL
   - Run: git branch --show-current → capture branch

2. Save collected settings to outputs/config.md:
   - AWS Account ID, Region, Profile
   - Git repository URL, branch
   - Any user-provided settings

3. Check docs/project-info.md (or project-info.template.md):
```

```
Check docs/project-info.md:
  → File exists AND has real content (not just template placeholders) → Use content
  → File missing OR template-only (contains "[" placeholders) → Switch to Q&A mode

Check docs/modernization-requirements.md:
  → File exists AND has real content → Use content
  → File missing OR template-only → Switch to Q&A mode for requirements
```

If Q&A mode is triggered, AI proactively asks for the needed information instead of waiting for the user to fill documents.

**Step 1: Check for project-info.md**
- If exists and has real content in `docs/project-info.md`:
  - Load Git repository URLs
  - **Clone immediately** to `source-code/` directory
  - Proceed to Step 3

**Step 2: If no project-info.md or template-only**
- AI proactively asks essential questions:
  ```
  I need the following information to start the analysis:

  1. Code repository URL: [required]
  2. Branch to analyze: [default: main]
  3. AWS Account ID: [required]
  4. AWS Region: [required]
  5. Database connection info: [if applicable]
  ```
- Clone to `source-code/[repo-name]/` directory

**Step 3: Analyze cloned code**
- Navigate to `source-code/[repo-name]/`
- Read package files (package.json, pom.xml, requirements.txt)
- Identify dependencies and versions
- Document configuration requirements
- Identify integration points

**Clone command**:
```bash
mkdir -p source-code
cd source-code
git clone [repo-url] [repo-name]
cd [repo-name]
git checkout [branch]
cd ../..
```

**Clone Verification (MANDATORY)**:
```bash
# Check .git directory
if [ ! -d "source-code/[repo-name]/.git" ]; then
  echo "❌ Git clone failed or incomplete"
  exit 1
fi

# Check package file exists
if [ ! -f "source-code/[repo-name]/package.json" ] && \
   [ ! -f "source-code/[repo-name]/pom.xml" ] && \
   [ ! -f "source-code/[repo-name]/requirements.txt" ]; then
  echo "❌ No package file found - clone may be incomplete"
  exit 1
fi

# Check source directories
if [ ! -d "source-code/[repo-name]/src" ] && \
   [ ! -d "source-code/[repo-name]/lib" ]; then
  echo "⚠️ No src/ or lib/ directory - verify clone completeness"
fi

echo "✅ Source code verification passed"
```

**If Verification Fails**:
1. Report to user immediately
2. Show error details
3. Request resolution (credentials, correct URL, network access)
4. **Do NOT proceed** with analysis
5. **Do NOT generate** analysis document with incomplete data
6. Wait for user to resolve issue
7. Retry clone after resolution

**Validation Checklist**:
- [ ] Git clone completed successfully
- [ ] .git directory exists
- [ ] Package file exists (package.json/pom.xml/requirements.txt)
- [ ] Source directories exist (src/ or lib/)
- [ ] Can read essential files

---

### Question Pre-Validation (MANDATORY)

Before writing any question in plan.md, AI must verify the question cannot be answered by direct investigation:

```
For each question you are about to write:
1. Can I answer this by reading source code? → Read it. Do NOT ask.
2. Can I answer this by running AWS CLI? → Run it. Do NOT ask.
3. Can I answer this by SSH/SSM into the server? → Try it. Do NOT ask.
4. None of the above can answer this? → Write the question in plan.md.
```

**Questions that SHOULD go to plan.md** (user-only knowledge):
- Business-specific context: peak traffic hours, seasonal patterns, SLA commitments
- Regulatory/compliance requirements: HIPAA, PCI-DSS, data residency
- Workload-specific operational conditions: maintenance windows, deployment freezes
- Future plans: expected growth, planned feature changes, team scaling
- External integration terms: partner API contracts, IP whitelisting agreements
- Organizational constraints: budget limits, team skill gaps, political considerations
- Non-functional requirements: acceptable downtime, performance targets

**Questions that MUST NOT go to plan.md** (AI can discover directly):
- Programming language, framework, version → Read source code
- Application port → Read config files, Dockerfile, source code
- Dependencies → Read package files
- EC2 instance count, type → `aws ec2 describe-instances`
- ALB existence and configuration → `aws elbv2 describe-load-balancers`
- Security group rules → `aws ec2 describe-security-groups`
- Running processes on server → SSH/SSM
- Environment variables on server → SSH/SSM
- Database engine and version → `aws rds describe-db-instances` or SSH/SSM

### Question Format
Use plan.md with [Question] and [Answer] tags:
```markdown
[Question]
What is the application port?

[Answer]
(User fills in)
```

### Proactive Follow-up Questions

After initial analysis, AI should proactively ask follow-up questions based on detected workload type:

**If web server detected**:
- "Is this serving static content, API, or both?"
- "What is the expected concurrent user count?"

**If worker/queue processor detected** (e.g., Celery, SQS consumer):
- "Are there long-running tasks? What is the maximum task duration?"
- "How does the worker handle SIGTERM? Is graceful shutdown implemented?"
- "Should queues be split into separate services for independent scaling?"

**If batch/scheduler detected**:
- "What is the schedule frequency?"
- "What happens if a batch job fails mid-execution? Is it idempotent?"

**If multiple processes in one codebase**:
- "Should these be separated into different containers, or kept together?"
- "How are they currently started? (e.g., shell script, supervisor, systemd)"

### AI Understanding Verification (MANDATORY)

After completing each analysis (application, database, or infrastructure), AI must present a structured summary of what it understood and ask the user to confirm:

```
## AI Understanding Verification

Based on my analysis, here is what I understood:

**Workload Summary**:
- [Type]: [Web server / Worker / Batch / Scheduler / etc.]
- [Environment]: [Development / Staging / Production]
- [Key characteristic]: [e.g., long-running Celery tasks, multi-process structure]

**Key Findings**:
1. [Finding 1]
2. [Finding 2]
3. [Finding 3]

**Constraints Identified**:
- [Constraint 1]
- [Constraint 2]

Please confirm:
- [ ] The summary above is correct
- [ ] Something is incorrect (please specify)
- [ ] Something is missing (please specify)
```

**CRITICAL**: Do NOT proceed to the next Stage until user confirms the summary is correct.

### Validation
- All required sections completed
- No assumptions without user confirmation
- Specific values (not "TBD" or "Unknown")
- AI Understanding Verification confirmed by user
- Ready for next stage
