# Getting Started Guide

This guide walks you through how to start workload modernization using the AI-Driven Modernization Prompt Sets.

## Prerequisites

### 1. Required Tools
- [ ] **Kiro CLI** installed (https://kiro.dev)
- [ ] **AWS CLI** configured (verify with `aws sts get-caller-identity`)
- [ ] **Code repository** accessible (Git clone ready)

### 2. Optional Tools (needed in Stage 5~6)
- [ ] Docker installed (for containerization)
- [ ] IaC tool: AWS CDK (`cdk --version`) or Terraform (`terraform --version`)
- [ ] DB connection info (for DB migration)

### 3. Input Document Preparation (Optional)
Preparing these documents in advance speeds up the analysis. **If not prepared, AI will collect the necessary information via Q&A.**

- `docs/project-info.md` — Project basic information (AWS account, region, code repository, etc.)
- `docs/modernization-requirements.md` — Modernization requirements

Refer to scenario-specific examples in the `examples/` directory for guidance.

### 4. Environment Setup

```bash
# Install Kiro CLI (macOS)
brew install kiro-cli
```

## Getting Started

```bash
# 1. Run Kiro CLI from the project directory
kiro-cli chat

# 2. Switch to the modernization agent
/agent swap modernization-agent

# 3. Type 'Let's start' and AI will guide you step by step
```

AI scans the `outputs/` directory to automatically detect current progress:
- **New start**: Checks docs/ files and guides from Stage 1
- **Resume**: Detects previous Stage outputs and proposes the next Stage

## Step-by-Step Flow

```
Stage 1: As-Is Analysis (Application, DB, Infrastructure)
  ↓
Stage 2: Requirement Analysis
  ↓
Stage 3: To-Be Architecture Design → Customer approval required
  ↓
Stage 4: Modernization Planning (choose ECS / EKS / DB)
  ↓
Stage 5: Pre-Execution Validation
  ↓
Stage 6: Task Execution
```

After each Stage is complete, AI automatically proposes the next Stage. Simply respond with "Please proceed."

### Context Reset Between Stages (Optional)

If conversation history from previous Stages has accumulated significantly, AI reasoning quality may degrade. If you notice response quality declining, we recommend resetting context with the `/clear` command.

```bash
/clear
```

After resetting, just let AI know and it will load the previous Stage's output documents to continue where you left off.

## Modernization Scenario Selection (Stage 4)

| Scenario | Best For |
|----------|----------|
| **Amazon ECS** | Simple container orchestration, AWS-native, quick start |
| **Amazon EKS** | Kubernetes standard, complex workloads, multi-cloud considerations |
| **Managed DB** | Data reliability first priority, Aurora/RDS migration |

## Example References

Check the `examples/` directory for `docs/` file writing examples:

- **E-Commerce Platform**: 3-tier web application modernization
- **SaaS Platform**: Background Worker optimization
- **Healthcare System**: AWS cloud modernization

## Manual Progression for Advanced Users

To start a specific Stage directly instead of auto-progression, paste the content of the corresponding prompt file into Kiro CLI:

```
prompts/1-as-is-analysis/application-analysis-prompt.md
prompts/2-requirement-analysis/requirement-analysis-prompt.md
prompts/3-to-be-architecture-design/architecture-design-prompt.md
prompts/4-make-task-lists/move-to-container-using-ecs-prompt.md
prompts/5-pre-execution-validation/1-automatic-validation-prompt.md
prompts/6-task-actions/ecs-path/start-move-to-container-using-ecs-prompt.md
```

## Important Notes

- ✅ Provide accurate information when AI asks
- ✅ Review and approve plans before execution
- ✅ Validate after each step completion
- ❌ Do not allow AI to make arbitrary assumptions
- ❌ Do not execute multiple stages at once

## Useful Kiro CLI Commands

| Command | Description |
|---------|-------------|
| `/agent swap modernization-agent` | Switch to modernization agent |
| `/clear` | Reset context (recommended between Stages) |
| `/help` | View help |
| `/quit` | Exit |

## Next Steps

- [Prompt Engineering Guide](.kiro/skills/common/prompt-engineering.md): How to write prompts
- [Modernization Guidelines](.kiro/skills/common/modernization-guidelines.md): Modernization principles
- [Example Projects](examples/): Various scenario references
