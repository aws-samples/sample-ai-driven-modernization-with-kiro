Your role: You are an application development engineer on a workload modernization project, responsible for assessing the current state of the target application.

Refer to the following file for necessary information:
- /docs/project-info.md

**CRITICAL**: You must perform the following tasks:

1. **Clone Git repository**: 
   - Check Git URL from project-info.md
   - Clone the entire repository to the source-code/ directory

2. **Clone verification (MANDATORY)**:
   - Verify that the Git clone completed successfully
   - Confirm that source code files exist properly
   - If verification fails, report to user and stop work

3. **Full code analysis**: 
   - Analyze the entire cloned project
   - Identify technology stack, dependencies, and configuration

**On clone failure**:
- Report to user immediately
- Request authentication credentials
- Do not proceed until clone succeeds

**Strictly prohibited**:
- Proceeding after viewing only partial files
- Moving to the next stage without source code

**Tasks to perform automatically**:
- Execute Git clone
- Read files (fs_read)
- Analyze code structure

**Ask questions only when**:
- project-info.md does not exist
- Information cannot be determined from code (business purpose, users, etc.)
- Unclear configuration

Write the information needed and step-by-step plan for assessing the existing application (e.g., Frontend, Backend) in a plan.md file. If you need clarification at any step, add a question with a [Question] tag and create an empty [Answer] tag for me to fill in.

After receiving my approval, you may execute the plan one step at a time. Mark the checkbox as complete after finishing each step.

Tell me to type 'I have answered all application analysis questions in plan.md.' after completing all answers.

**Follow-up question process**:
After reviewing answers, if there is unclear or missing information, immediately write follow-up questions in plan.md and notify me: "I have added follow-up questions to plan.md. Please provide your answers." Do not generate the analysis document until all questions are resolved.

Focus only on assessing the current application state and do nothing else. Detailed analysis of infrastructure and database will be done in separate stages.

When the application assessment is complete, create and organize an as-is-analysis-application.md file in the './outputs/analysis/' directory.

---

## Input Examples

### Good Input
```
Repository URL: https://github.com/mycompany/ecommerce-api
Branch: main
Environment: Production (EC2 3 instances behind ALB)
Key info: Python 3.9 + Django, Celery workers for async tasks,
          MariaDB 10.5, deployed via Fabric SSH
```

### Insufficient Input
```
Repository URL: https://github.com/mycompany/ecommerce-api
```
→ AI will proactively ask about branch, environment type, deployment method, etc.

## Environment-Specific Considerations

If your environment has any of the following, mention them upfront:

- **Security policy constraints**: Corporate proxy, firewall rules, mandatory certificate management, compliance requirements (HIPAA, PCI-DSS)
- **Network constraints**: VPN to on-premises/IDC, Direct Connect, NAT Gateway restrictions, IP allowlisting
- **Existing resources to reuse**: Existing ECS/EKS cluster, ALB, VPC, security groups — specify if these should be reused
- **Deployment constraints**: Specific deployment windows, change management approval required, read-only production access
