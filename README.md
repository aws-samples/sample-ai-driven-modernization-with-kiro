# AI-Driven Modernization Prompt Sets

A systematic collection of prompts based on AI-driven modernization methodology, designed to transform legacy workloads into modern architectures built on AWS managed services.

## Overview

This project is a prompt set designed to guide the step-by-step modernization of legacy workloads in collaboration with an AI agent. It systematically covers the modernization methodology from analysis to planning and task execution, and prompts can be customized to fit the target workload's requirements and context.

> 💡 This prompt set provides a **starting point and guidelines** for the modernization process. Actual modernization outcomes may vary depending on workload complexity, team capabilities, and infrastructure environment. We recommend adjusting prompts with expert review at each stage.

**Modernization Scenarios**:
- Container orchestration (Amazon ECS / EKS)
- Managed database migration (Aurora / RDS)
- Infrastructure as Code (IaC)
- CI/CD pipeline setup

**Key Objectives**:
- Improve availability
- Traffic handling via Auto Scaling
- Ensure data reliability
- Operational efficiency

## Quick Start

### Prerequisites

**Required Tools**:
- [Kiro CLI](https://kiro.dev) installed (`brew install kiro-cli`)
- AWS CLI configured (verify with `aws sts get-caller-identity`)
- Source code accessible (Git clone ready)

**Optional Tools** (needed in Stage 5~6):
- Docker installed (for containerization)
- IaC tool: AWS CDK (`cdk --version`) or Terraform (`terraform --version`)
- DB connection info (for DB migration)

### Getting Started
```bash
# 1. Clone the project
git clone <repository-url>
cd ai-driven-modernization-prompt-sets

# 2. Prepare input documents (optional — AI will collect via Q&A if not prepared)
#    Refer to examples/ directory for scenario-specific writing examples
cp docs/project-info.template.md docs/project-info.md
cp docs/modernization-requirements.template.md docs/modernization-requirements.md

# 3. Run Kiro CLI
kiro-cli chat

# 4. Switch to the modernization agent
/agent swap modernization-agent

# 5. Type 'Let's start' and AI will guide you step by step
```

For details, see the [Getting Started Guide](GETTING_STARTED.md).

## Prompt Workflow

### Stage 1: As-Is Analysis
`/prompts/1-as-is-analysis/`

Assess the current state of the application, database, and infrastructure.

### Stage 2: Requirement Analysis
`/prompts/2-requirement-analysis/`

Organize modernization requirements and set priorities.

### Stage 3: To-Be Architecture Design
`/prompts/3-to-be-architecture-design/`

Design the To-Be architecture based on As-Is analysis and requirements. Includes AWS best practice-based design, Compliance Matrix, and production cutover strategy. Proceeds to the next stage after customer approval.

### Stage 4: Modernization Planning
`/prompts/4-make-task-lists/`

Choose from 3 modernization scenarios:
- **Amazon ECS-based containerization**
- **Amazon EKS-based containerization**
- **Managed database migration**

### Stage 5: Pre-Execution Validation
`/prompts/5-pre-execution-validation/`

Automated tool and access testing, risk analysis. Proactively identifies potential issues in the task list.

### Stage 6: Task Execution
`/prompts/6-task-actions/`

AI agent executes the task list sequentially.

## .kiro/ Structure

Configuration files that control AI agent behavior.

```
.kiro/
├── steering/       # Rules & routing (always loaded)
│   ├── workflow-orchestration.md   # Stage-specific guide loading directives
│   └── audit-rules.md             # Audit logging rules
├── guides/         # Safety-critical guides (explicitly loaded per Stage)
│   ├── stages/          # Stage-specific execution methodology
│   ├── aws-practices/   # Quality gates, readiness checks
│   └── common/          # Execution rules, output templates
├── skills/         # Reference guides (loaded on-demand based on conversation context)
│   ├── technical/       # Containers, IaC, DB migration, testing
│   ├── aws-practices/   # Strategy, ECS/EKS selection, traffic cutover
│   └── common/          # Directory structure, risk discovery methodology
├── settings/       # CLI and agent settings
│   └── cli.json                   # CLI configuration
└── agents/
    └── modernization-agent.json   # Agent configuration
```

- **steering**: Rules applied to every conversation. Directs which safety-critical guides to load for each Stage
- **guides**: Guides that would cause safety issues if skipped. Explicitly loaded per steering directives
- **skills**: Reference materials needed only in specific situations. AI automatically loads relevant skills based on conversation context

## Output Structure

```
/outputs/
├── analysis/          # As-is and requirement analysis results
├── plan/              # Modernization plan
├── validation/        # Pre-execution validation report
└── tasks/             # Step-by-step task lists
    ├── ecs/
    ├── eks/
    └── database/
```

## Example Projects

The `examples/` directory provides **scenario-specific examples** of how to write input documents (`project-info.md`, `modernization-requirements.md`) in `docs/`:

- **E-Commerce Platform**: 3-tier → Containers + Microservices
- **SaaS Platform**: Background Worker modernization
- **Healthcare System**: AWS cloud modernization

See the [examples/](examples/) directory for details.

## Key Documents

- [Getting Started Guide](GETTING_STARTED.md): How to start the project
- [Project Info Template](docs/project-info.template.md): Project information writing guide
- [Requirements Template](docs/modernization-requirements.template.md): Requirements writing guide

## Technology Stack

- **Containers**: Docker, Amazon ECR
- **Orchestration**: Amazon ECS / Amazon EKS
- **Database**: Amazon Aurora / Amazon RDS
- **IaC**: AWS CDK / Terraform
- **CI/CD**: AWS CodePipeline, AWS CodeBuild, AWS CodeDeploy
- **Monitoring**: Amazon CloudWatch

## Design Principles

- **Sequential execution**: Analysis → Design → Implementation
- **Step-by-step validation**: Immediate validation after each task completion
- **Production quality**: Security, high availability, cost optimization considered
- **Data integrity**: Backup required before migration, rollback plan established

## Contributing

This project was created to share various modernization scenarios. Please contribute if you have new examples or improvements.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
