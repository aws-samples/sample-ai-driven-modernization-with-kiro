---
name: directory-structure
description: Project directory structure rules. Path rules for source-code/, infrastructure/, outputs/. Dockerfile/IaC file creation locations, working directory rules. Reference when creating/navigating files.
---
# Directory Structure Guide

## Purpose
Define standard directory structure for modernization projects.

---

## Project Layout

```
<workspace-root>/
│
├── source-code/                    # Source code (cloned)
│   ├── backend/                    # Backend repository
│   │   ├── src/
│   │   ├── tests/
│   │   ├── package.json            # Dependencies
│   │   └── Dockerfile              # ← Create here
│   │
│   └── frontend/                   # Frontend repository (if applicable)
│       ├── src/
│       ├── package.json
│       └── Dockerfile              # ← Create here
│
├── infrastructure/                 # IaC code (generated)
│   ├── bin/
│   │   └── app.ts
│   ├── lib/
│   │   ├── network-stack.ts
│   │   ├── container-stack.ts
│   │   └── database-stack.ts
│   ├── cdk.json
│   └── package.json
│
└── outputs/                        # All output documents
    ├── analysis/
    │   ├── as-is-analysis-application.md
    │   ├── as-is-analysis-database.md
    │   ├── as-is-analysis-infrastructure.md
    │   └── requirements-analysis.md
    │
    ├── plan/
    │   └── modernization-plan-[ecs|eks|database].md
    │
    └── tasks/
        ├── ecs/
        │   ├── 1. Foundation Infrastructure Tasks.md
        │   ├── 2. Application Containerization Tasks.md
        │   └── ...
        ├── eks/
        └── database/
```

---

## Path Rules

### Source Code
- **Clone to**: `source-code/[repo-name]/`
- **Examples**: 
  - `source-code/backend/`
  - `source-code/frontend/`
  - `source-code/myapp/` (monorepo)
- **Reason**: Keep original code separate and organized

### Dockerfile
- **Create in**: `source-code/[repo-name]/Dockerfile`
- **Examples**:
  - `source-code/backend/Dockerfile`
  - `source-code/frontend/Dockerfile`
- **Reason**: Dockerfile belongs with the code it containerizes
- **Build context**: Same directory as Dockerfile

### Infrastructure Code
- **Create in**: `infrastructure/`
- **Examples**:
  - `infrastructure/lib/network-stack.ts`
  - `infrastructure/lib/container-stack.ts`
- **Reason**: Separate infrastructure from application code
- **Working directory**: `infrastructure/` when running CDK/Terraform

### Output Documents
- **Create in**: `outputs/[category]/`
- **Categories**:
  - `analysis/`: All analysis documents
  - `plan/`: Modernization plans
  - `tasks/`: Task list files
- **Reason**: Keep all outputs organized and separate from code

---

## Repository Structure Variations

### Multi-repo (Separate repositories)
```
source-code/
├── backend/          # Separate repo
└── frontend/         # Separate repo
```

### Monorepo (Single repository)
```
source-code/
└── myapp/            # One repo
    ├── backend/
    └── frontend/
```

### Single Application
```
source-code/
└── myapp/            # One repo, one app
    └── src/
```

**AI adapts to actual structure found**

---

## Working Directory Rules

### For Analysis
- Start: `<workspace-root>`
- Navigate to: `source-code/[repo-name]/` to analyze code
- Return to: `<workspace-root>` after analysis

### For Containerization
- Start: `<workspace-root>`
- Navigate to: `source-code/[repo-name]/` for each app
- Create: `Dockerfile` in current directory
- Build: `docker build -t myapp .` (in same directory)
- Return to: `<workspace-root>` after build

### For IaC Development
- Start: `<workspace-root>`
- Navigate to: `infrastructure/` (create if doesn't exist)
- Create: Stack files in `lib/` subdirectory
- Execute: `cdk deploy` from `infrastructure/` directory
- Return to: `<workspace-root>` after deployment

### For Task Execution
- Start: `<workspace-root>`
- Navigate as needed for each task
- Always return to `<workspace-root>` between tasks

---

## File Creation Rules

### Never Create In
- ❌ `outputs/` - No code files
- ❌ `source-code/` root - Only in repo subdirectories
- ❌ `.kiro/` - Only steering/guides

### Always Create In
- ✅ `source-code/[repo]/Dockerfile` - Dockerfiles
- ✅ `infrastructure/lib/*.ts` - IaC code
- ✅ `outputs/analysis/*.md` - Analysis docs
- ✅ `outputs/plan/*.md` - Plans
- ✅ `outputs/tasks/*/*.md` - Task lists

---

## Validation

### Directory Structure Check
Before starting work:
- [ ] `source-code/` exists with cloned repositories
- [ ] `outputs/` exists for documents
- [ ] `infrastructure/` will be created when needed

### Path Verification
During work:
- [ ] Creating files in correct locations
- [ ] Not mixing code and documentation
- [ ] Following working directory rules
