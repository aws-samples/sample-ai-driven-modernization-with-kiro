---
name: risk-discovery-methodology
description: Risk discovery methodology before modernization execution. Cross-references as-is analysis, requirements, and task lists to generate context-aware risk checkpoints. Reference during Stage 4 pre-execution validation interactive review (Part 2).
---

# Risk Discovery Methodology

## Purpose
Discover and recommend potential risk checkpoints based on the customer's actual analysis documents and task lists before modernization execution.

---

## How to Use

### Step 1: Load Documents
Read all of the following documents:
- `outputs/analysis/as-is-analysis-application.md`
- `outputs/analysis/as-is-analysis-database.md` (if applicable)
- `outputs/analysis/as-is-analysis-infrastructure.md`
- `outputs/analysis/requirements-analysis.md`
- `outputs/tasks/*/*.md` (all task lists)

### Step 2: Check Category Triggers
Check trigger conditions for each category against the documents. Skip categories that do not match any triggers.

### Step 3: Generate Questions
Replace `{variables}` in question templates with actual analysis data to generate specific checkpoints.

### Step 4: Recommend
Organize generated checkpoints by category and recommend to the user.

---

## AI Tone Guide

Do not ask questions directly. Present points worth asking about in a **recommendation format**.

**Pattern**:
```
"{fact discovered from analysis documents} was identified.
{potential risk description} could occur,
so it might be worth asking whether {specific checkpoint} has been considered."
```

**Good example**:
✅ "The as-is analysis identified a payment gateway API integration. If this API uses server IP-based authentication, the integration could break when transitioning to ECS since the IP would change to a NAT Gateway IP. It might be worth asking whether diagnosing and addressing this issue has been considered."

**Bad example**:
❌ "Does the payment gateway API use IP-based authentication?" (direct question)
❌ "Are there any external integration systems?" (re-asking what was already identified in analysis)

---

## Category Triggers and Question Templates

### 1. Common Risks

**Trigger**: Always (applies to all modernization projects)
**Check against**: Downtime tolerance in requirements, rollback plans, validation steps in task lists

Question templates:
- "The requirements specify a downtime tolerance of {downtime tolerance}. It might be worth checking whether the estimated duration of each step in the task list, when summed up, falls within this window."
- "There appears to be a dependency between {Phase N} and {Phase M} in the task list. It might be worth checking whether the impact on {Phase M} if {Phase N} fails, and the corresponding mitigation plan, have been considered."
- "The task list includes validation steps, but it might be worth checking whether specific pass/fail criteria (e.g., abort if error rate exceeds X%) have been defined for validation failures."

---

### 2. Business Continuity

**Trigger**: When peak traffic, batch jobs, SLA, or deployment time constraints are mentioned in requirements
**Check against**: Operational metrics, downtime tolerance, deployment time constraints, business calendar

Question templates:
- "The requirements mention {peak hours/events}. It might be worth checking whether the transition schedule conflicts with this period, and whether the new environment can handle the traffic when {peak hours} arrive immediately after the transition."
- "The as-is analysis identified {batch jobs/scheduled tasks}. It might be worth checking whether these jobs will continue to run normally during the transition, and whether there is any risk of the transition timing overlapping with batch execution."
- "There are flows like {orders/payments} that require transaction integrity. It might be worth asking whether measures to prevent in-flight transactions from being lost or duplicated at the cutover point are included in the task list."
- "The requirements limit deployment time to {deployment time constraint}. It might be worth checking whether there are dates to avoid on the business calendar, such as month-end settlements or quarterly reporting periods."

---

### 3. Security / Compliance

**Trigger**: When authentication/authorization logic, sensitive data handling, or compliance requirements are found in as-is analysis
**Check against**: Environment variables, config files, security groups, IAM, encryption settings

Question templates:
- "The as-is analysis identified sensitive information (DB passwords, API keys, etc.) in {config files/environment variables}. It might be worth checking whether tasks to securely migrate this information to Secrets Manager or Parameter Store are included."
- "The system currently uses {authentication method}. It might be worth checking whether this authentication method will work as-is in a container/managed service environment, and whether network path changes could break existing firewall/ACL rules."
- "The requirements specify {compliance requirements}. It might be worth checking whether audit log continuity is maintained after modernization, and whether log retention policies meet regulatory requirements."
- "The current encryption configuration for data in transit/at rest is {current encryption status}. It might be worth checking whether the same level of encryption is applied to new resources (ALB, ECS, RDS, etc.) in the task list."

---

### 4. External Integrations / Third-Party

**Trigger**: When external APIs or third-party services are found in Integration Points of as-is analysis
**Check against**: Integration system list, authentication methods, endpoints, licenses

Question templates:
- "{External API/service name} integration was identified. If this API uses server IP-based authentication, the integration could break when the IP changes during infrastructure transition. It might be worth asking whether diagnosing and addressing this issue has been considered."
- "If the integration with {external service name} uses mTLS certificates, it might be worth checking whether the certificates will remain valid in the new environment and whether the certificate renewal process will change."
- "{Third-party SDK/library} is in use. It might be worth checking whether there are compatibility issues in a container environment (especially Alpine Linux-based), and whether the licensing billing model changes."
- "It might be worth checking whether there are contractual requirements to notify external partners of endpoint or IP changes in advance, and whether the notification lead time is reflected in the transition schedule."

---

### 5. Data Consistency / State Management

**Trigger**: When session management, cache, file uploads, or event queues are found in as-is analysis
**Check against**: Session storage method, cache strategy, file storage, message queues

Question templates:
- "The as-is analysis identified {session storage method}. In a container environment, instances are dynamically created/destroyed, so sessions could be lost. It might be worth asking whether migrating sessions to an external store (Redis, DynamoDB, etc.) has been considered."
- "There may be in-flight {transactions/requests} at the cutover point. It might be worth checking whether a graceful shutdown or draining strategy to prevent these requests from being lost or duplicated is included in the task list."
- "{Cache system} is in use. After the transition, the cache will be empty (cold start), which could cause a sudden load spike on the database. It might be worth checking whether a cache warm-up strategy has been considered."
- "The as-is analysis identified {file uploads/local storage} usage. Due to the ephemeral storage nature of containers, files could be lost. It might be worth checking whether tasks to migrate to external storage such as S3 are included."
- "It might be worth asking whether measures to prevent unprocessed messages in {message queue/event system} from being lost during the transition have been considered."

---

### 6. Containerization-Specific

**Trigger**: When containerization (ECS/EKS) related Phases exist in task lists
**Check against**: Dockerfile, environment variables, ports, Health Check, dependencies

Question templates:
- "The as-is analysis identified that the application runs on {port} and uses {environment variable list}. It might be worth checking whether tasks to convert these settings to external injection methods in the container environment are included."
- "The application currently depends on {local file path}. This path may not exist in a container environment. It might be worth checking whether tasks to resolve this dependency have been considered."
- "The Health Check endpoint is currently {current status}. A proper Health Check is essential for the container orchestrator to accurately determine container health. It might be worth checking whether this is included in the task list."
- "The application startup time is {estimated time}. During container scaling, requests cannot be served during this period. It might be worth asking whether Readiness Probe configuration or minimum instance count adjustments have been considered."
- "It might be worth checking whether multi-architecture builds (amd64 + arm64) are included in the task list. Using Graviton instances can reduce costs by up to 40%."

---

### 7. DB Migration-Specific

**Trigger**: When database migration related Phases exist in task lists
**Check against**: DB engine, schema complexity, Stored Procedures, data size

Question templates:
- "The as-is analysis identified {number of Stored Procedures/Triggers}. It might be worth checking whether pre-validation tasks to verify compatibility with the target DB ({target DB engine}) are included."
- "The database size is {DB size}. It might be worth verifying whether the expected migration duration for this scale falls within the downtime tolerance ({downtime tolerance})."
- "Write operations will continue on the source DB during migration. It might be worth checking whether a CDC (Change Data Capture) strategy to synchronize changes is included in the task list."
- "It might be worth checking whether record count comparison alone is sufficient for data validation, or whether additional verification such as checksum comparison or sample data comparison is needed."
- "The application's DB connection string currently uses {current connection method}. It might be worth checking whether connection pool sizing also needs to be readjusted alongside the task to switch to the target DB endpoint."

---

### 8. Infrastructure / Network

**Trigger**: When VPC, subnet, security group, ALB or other infrastructure changes exist in task lists
**Check against**: Network configuration, security group rules, DNS, routing

Question templates:
- "The current {security group configuration} was identified. It might be worth checking whether all required ports and communication paths for the new architecture are reflected in the security groups, and conversely, whether there are any unnecessarily open ports."
- "When switching DNS, the current TTL value is {current TTL}. It might be worth checking whether lowering the TTL before the transition to minimize DNS cache-related issues has been considered."
- "Multi-AZ deployment is included in the requirements, and the subnets are configured as {current AZ configuration}. It might be worth checking whether inter-AZ communication latency and data transfer costs have been considered in the new architecture."
- "There is currently a {VPN/Direct Connect} connection. It might be worth asking whether tasks to verify that this connection works properly after the infrastructure change are included."

---

### 9. Cost Pitfalls

**Trigger**: Always (especially when infrastructure changes are included)
**Check against**: Cost constraints in requirements, current resource sizing, new architecture configuration

Question templates:
- "The new architecture includes a NAT Gateway. NAT Gateway data processing costs can amount to significant monthly expenses depending on traffic volume. It might be worth estimating the expected costs."
- "Multi-AZ deployment incurs inter-AZ data transfer costs. Given {expected traffic volume}, it might be worth checking whether these costs are significant."
- "If there are existing Reserved Instances or Savings Plans in use, it might be worth checking whether these commitments would be wasted after modernization."
- "The billing model for {third-party license} may change in a container environment (per-server to per-container). It might be worth asking whether license cost changes have been verified."
- "If dev/staging environments are configured with the same architecture, additional costs will be incurred. It might be worth checking whether cost optimization measures (nighttime shutdown, reduced configuration, etc.) have been considered."

---

### 10. Operations / Monitoring

**Trigger**: When monitoring, logging, or alarm related Phases exist in task lists
**Check against**: Current monitoring tools, log format, alarm settings

Question templates:
- "The system currently uses {monitoring tool/method}. It might be worth checking whether existing monitoring covers the new environment after modernization, or whether new monitoring setup is needed."
- "Log formats may change before and after modernization. If there are existing log parsing systems (ELK, Splunk, etc.), it might be worth checking whether parsing rule updates are needed."
- "Collecting a performance baseline (response time, throughput, error rate) of the current system before modernization would serve as a comparison reference after the transition. It might be worth checking whether this baseline collection task is included."
- "It might be worth asking whether tasks to configure alerts in the new environment to be delivered to appropriate channels (Slack, PagerDuty, etc.) when incidents occur are included."

---

### 11. Team / Organizational Readiness

**Trigger**: Always (especially when architecture changes significantly)
**Check against**: Team capabilities in requirements, current operational methods

Question templates:
- "When an incident occurs in the new environment (ECS/EKS, Aurora, etc.) after modernization, it might be worth asking whether the operations team has the capability to troubleshoot, and whether pre-training or runbook preparation has been considered."
- "It might be worth checking whether there is a communication plan for issues on the transition day, for example, who makes the rollback decision and how customers are notified."
- "It might be worth checking whether stakeholders (executives, clients, etc.) have been informed about potential SLA violation risks, and whether SLA exceptions during the transition period are needed."
- "It might be worth checking whether on-call personnel know how to check logs, restart containers, and adjust scaling in the new environment."

---

### 12. Rollback Scenarios

**Trigger**: Always (especially when DB migration or production deployment is included)
**Check against**: Rollback plans in task lists, identification of irreversible changes

Question templates:
- "If the task list includes DB schema changes, it might be worth checking whether these changes are forward-only (irreversible), or whether the schema can be rolled back together during a rollback."
- "Once data starts being written to the new system during the transition, it might be worth asking whether there is a plan for how to handle this data during a rollback (discard it, or sync it back to the old system)."
- "It might be worth verifying whether partial rollback is possible, for example, whether a scenario where only the backend is rolled back while the frontend is maintained would work."
- "It might be worth checking whether rollback decision criteria (e.g., error rate exceeding X%, response time exceeding Xms) and the decision-maker have been clearly defined."
- "It might be worth checking whether an actual rollback rehearsal has been conducted. Attempting a rollback in production without a rehearsal could lead to unexpected issues, so it might be worth asking whether a rehearsal plan is included."
