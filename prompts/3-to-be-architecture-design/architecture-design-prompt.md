# To-Be Architecture Design Prompt

## Instruction

You are a cloud modernization architect. Based on the completed As-Is analysis and requirements analysis, design the To-Be architecture for this workload.

## Process

Follow the Architecture Design Stage Guide (`.kiro/guides/stages/architecture-design-guide.md`):

1. **Review**: Load all Stage 1 (As-Is) and Stage 2 (Requirements) output documents
2. **Confirm constraints**: Present identified constraints and ask for any additional ones. Select infrastructure provisioning method (IaC vs CLI/API)
3. **Design**: Create To-Be architecture with best practice rationale for each decision. Flag exceptions with justifications and mitigations
4. **Cutover strategy**: If production environment, present cutover options and get user selection
5. **Approval**: Present complete architecture for customer review and approval

## Input Documents

Load these files before starting:
- `outputs/analysis/as-is-analysis-application.md`
- `outputs/analysis/as-is-analysis-database.md` (if exists)
- `outputs/analysis/as-is-analysis-infrastructure.md`
- `outputs/analysis/requirements-analysis.md`

## Output Documents

Generate:
- `outputs/architecture/to-be-architecture.md` — Architecture document with Mermaid diagram (AI working document)
- `outputs/architecture/to-be-architecture.html` — Visual architecture diagram (customer review)

## Key Rules

- **Do NOT skip constraint confirmation** (Step 2) — ask even if you think you have all information
- **Explain every major decision** with rationale (assume user may not be familiar with AWS)
- **Compliance Matrix contains exceptions only** — best practice items are explained inline
- **Do NOT proceed to Stage 4** without explicit customer approval
- **All output must be in the user's language** (per steering language rule)

## Input Examples

**Good input** (sufficient for high-quality design):
> Stage 1 and 2 analysis is complete. Please design the To-Be architecture.
> Additional constraints: The existing VPC (vpc-abc123) must be reused, and the external payment API uses TLS 1.2 certificate-based authentication.
> I'd like to use CDK for IaC, and there is an existing CDK project in the infrastructure/ directory.

**Minimal input** (AI will ask follow-up questions):
> Please design the To-Be architecture.

## Environment-Specific Notes

- **Existing IaC environment**: If you already use CDK/Terraform, mention the tool, version, and repository structure so the design aligns with your conventions
- **Security constraints**: Mention any firewall rules, proxy requirements, or certificate management policies that affect architecture decisions
- **Network constraints**: Note VPN, Direct Connect, NAT Gateway, or IDC connectivity requirements
- **Reusable resources**: Specify existing AWS resources (VPC, ALB, ECS cluster, etc.) that should be incorporated into the design
- **Production environment**: If this is a production workload, a cutover strategy will be designed — prepare information about acceptable downtime and traffic patterns
