Your role: You are an infrastructure engineer on a workload modernization project, responsible for assessing the current state of the target infrastructure.

Refer to the following file for necessary information:
- /docs/project-info.md

**CRITICAL**: You must perform the following tasks:
1. **Run AWS CLI**: Check AWS account info from project-info.md and query resources
2. **Resource analysis**: Verify actual resources such as EC2, RDS, VPC, Security Groups
3. **Automated collection**: Automatically collect resource information via AWS CLI

**Tasks to perform automatically**:
- Execute `aws ec2 describe-instances`
- Execute `aws rds describe-db-instances`
- Execute `aws elbv2 describe-load-balancers`
- Collect actual resource information (using AWS CLI or AWS tools)

**Ask questions only when**:
- project-info.md does not exist
- AWS CLI access is unavailable
- Information cannot be determined from resources

**Server internals (auto-collect if SSH accessible, ask if not)**:
- Server OS type and version
- Compute resource specs (CPU, Memory, Storage, Network I/O)
- Running process list (agents, sidecars, cron jobs beyond the main application)
- Key configuration file list and locations (nginx, supervisor, logrotate, etc.)
- System-level installed packages/libraries
- Log collection method (file-based, CloudWatch Agent, Fluentd, etc.)

Write the information needed and step-by-step plan for assessing the existing infrastructure in a plan.md file. If you need clarification at any step, add a question with a [Question] tag and create an empty [Answer] tag for me to fill in.

After receiving my approval, you may execute the plan one step at a time. Mark the checkbox as complete after finishing each step.

Tell me to type 'I have answered all infrastructure analysis questions in plan.md.' after completing all answers.

**Follow-up question process**:
After reviewing answers, if there is unclear or missing information, immediately write follow-up questions in plan.md and notify me: "I have added follow-up questions to plan.md. Please provide your answers." Do not generate the analysis document until all questions are resolved.

Focus only on assessing the current infrastructure state and do nothing else. Detailed analysis of application and database will be done in separate stages.

When the infrastructure assessment is complete, create and organize an as-is-analysis-infrastructure.md file in the './outputs/analysis/' directory.

---

## Input Examples

### Good Input
```
AWS Account: 123456789012, ap-northeast-2
Infra: 3x EC2 (c5.xlarge) behind ALB, single AZ
Network: VPC with public/private subnets, NAT Gateway, VPN to IDC
Security: WAF on ALB, Security Groups per tier
Existing resources to reuse: VPC, ALB, existing ECS cluster
```

### Insufficient Input
```
AWS Account: 123456789012
```
→ AI will proactively ask about region, compute resources, network setup, security, reusable resources, etc.

## Environment-Specific Considerations

If your environment has any of the following, mention them upfront:

- **Security policy constraints**: Corporate proxy, firewall rules, mandatory certificate management, compliance requirements
- **Network constraints**: VPN to on-premises/IDC, Direct Connect, NAT Gateway restrictions, IP allowlisting
- **Existing resources to reuse**: Existing ECS/EKS cluster, ALB, VPC, security groups
- **Multi-account setup**: Landing Zone, Control Tower, cross-account access patterns
