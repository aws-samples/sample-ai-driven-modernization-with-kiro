# Workflow Orchestration Guide

## Purpose
Master guide that directs AI to load appropriate steering files for each stage.

**Note**: Kiro CLI automatically loads all files in `.kiro/steering/`. This file helps AI understand which guides to reference for each stage.

**Skills**: Technical and reference guides are registered as skills in `.kiro/skills/`. AI automatically loads relevant skills based on conversation context. Only **safety-critical guides (must-load guides)** are listed below.

---

## Stage 1: As-Is Analysis

### When
User provides one of these prompts:
- `prompts/1-as-is-analysis/application-analysis-prompt.md`
- `prompts/1-as-is-analysis/database-analysis-prompt.md`
- `prompts/1-as-is-analysis/infrastructure-analysis-prompt.md`

### Load These Guides (Safety-Critical)
- `.kiro/guides/stages/analysis-stage-guide.md` - Analysis methodology and checklists
- `.kiro/guides/common/output-templates.md` - Output format templates

### Execution
1. **MANDATORY**: Log start of analysis in `outputs/audit.md` with timestamp
2. Follow analysis-stage-guide.md methodology
3. Create plan.md with questions
4. Wait for user answers
5. Execute analysis
6. Generate output document
7. **MANDATORY**: Log completion and output files in `outputs/audit.md`

### Expected Output
- `outputs/analysis/as-is-analysis-[application|database|infrastructure].md`

### Quality Check
- Use checklist from analysis-stage-guide.md
- Ensure all required sections completed
- No assumptions without user confirmation

---

## Stage 2: Requirement Analysis

### When
User provides:
- `prompts/2-requirement-analysis/requirement-analysis-prompt.md`

### Load These Guides (Safety-Critical)
- `.kiro/guides/stages/requirement-analysis-guide.md` - Requirements gathering process
- `.kiro/guides/common/output-templates.md` - Output format

### Execution
1. **MANDATORY**: Log start of requirement analysis in `outputs/audit.md` with timestamp
2. Review all as-is analysis documents
3. Create plan.md with requirement questions
4. Wait for user answers
5. Analyze requirements and generate output
6. **MANDATORY**: Log completion and output files in `outputs/audit.md`

### Expected Output
- `outputs/analysis/requirements-analysis.md`

### Quality Check
- Include Five Lenses assessment questions
- Recommend modernization pattern (Rehost/Replatform/Refactor/etc.)
- Define cloud-native target characteristics (if Refactor/Replatform)
- Ensure NFRs are specific and measurable

---

## Stage 3: To-Be Architecture Design

### When
User provides:
- `prompts/3-to-be-architecture-design/architecture-design-prompt.md`

### Load These Guides (Safety-Critical)
- `.kiro/guides/stages/architecture-design-guide.md` - Architecture design process (5 Steps)
- `.kiro/guides/common/output-templates.md` - Output format templates

### Execution
1. **MANDATORY**: Log start of architecture design in `outputs/audit.md` with timestamp
2. Load all Stage 1 and Stage 2 output documents
3. Follow architecture-design-guide.md 5-Step process:
   - Step 1: Review all analysis documents
   - Step 2: Confirm constraints + select provisioning method (IaC vs CLI)
   - Step 3: Design To-Be architecture with best practice rationale
   - Step 4: Production cutover strategy (if production environment)
   - Step 5: Customer review and approval
4. **MANDATORY**: Do NOT proceed to Stage 4 without explicit customer approval
5. **MANDATORY**: Log completion and output files in `outputs/audit.md`

### Expected Output
- `outputs/architecture/to-be-architecture.md` (AI working document, Mermaid)
- `outputs/architecture/to-be-architecture.html` (Customer review, styled Mermaid)

### Quality Check
- Architecture diagram complete (all components, data flow, integration points)
- Each major decision has inline rationale
- Compliance Matrix contains only justified exceptions
- Cutover strategy defined (if production)
- Provisioning method selected and documented
- Customer approval obtained

---

## Stage 4: Make Task Lists

### When
User provides one of these prompts:
- `prompts/4-make-task-lists/move-to-container-using-ecs-prompt.md`
- `prompts/4-make-task-lists/move-to-container-using-eks-prompt.md`
- `prompts/4-make-task-lists/move-to-managed-database-prompt.md`

### Load These Guides (Safety-Critical)
- `.kiro/guides/stages/planning-stage-guide.md` - Planning methodology and task structure
- `.kiro/guides/aws-practices/readiness-checklist.md` - Prerequisites verification
- `.kiro/guides/common/output-templates.md` - Task file format

### Prerequisites
- **CRITICAL**: Stage 3 (To-Be Architecture Design) must be approved
- Load approved `outputs/architecture/to-be-architecture.md` as input

### Before Generating Tasks
**CRITICAL**: Perform readiness check
1. **MANDATORY**: Log start of planning in `outputs/audit.md` with timestamp
2. Load `.kiro/guides/aws-practices/readiness-checklist.md`
3. Verify all prerequisites
4. Identify blockers and gaps
5. If blockers: Report and wait for resolution
6. If gaps: Ask user how to proceed
7. Only generate tasks when ready
8. **MANDATORY**: Log completion and output files in `outputs/audit.md`

### Expected Output
- `outputs/plan/modernization-plan-[ecs|eks|database].md`
- `outputs/tasks/[ecs|eks|database]/[1-9]. [Phase Name] Tasks.md`

### Quality Check
- Use checklist from planning-stage-guide.md
- Ensure readiness check passed
- Verify all phases covered
- Confirm tasks are actionable and specific
- Tasks align with approved To-Be architecture

---

## Stage 5: Pre-Execution Validation

### When
User provides one of these prompts:
- `prompts/5-pre-execution-validation/1-automatic-validation-prompt.md`
- `prompts/5-pre-execution-validation/2-risk-analysis-container-prompt.md`
- `prompts/5-pre-execution-validation/2-risk-analysis-database-prompt.md`

### Load These Guides (Safety-Critical)
- `.kiro/guides/aws-practices/readiness-checklist.md` - Comprehensive checklist

### Validation Actions

**Stage 5 consists of TWO mandatory steps. Both MUST be completed before proceeding to Stage 6.**

**Step 1 (MANDATORY): Automatic Validation — Tool & Access Testing**
AI must execute:
1. **MANDATORY**: Log start of validation in `outputs/audit.md` with timestamp
2. Test Git access: `git ls-remote [repo-url]`
3. Test AWS CLI: `aws sts get-caller-identity`
4. Check Docker: `docker --version`
5. Check IaC tool: `cdk --version` or `terraform --version` (if IaC selected in Stage 3)
6. Test DB connection: `mysql -h [host] -e "SELECT 1"` (if applicable)
7. **MANDATORY**: Log validation results in `outputs/audit.md`
8. **MANDATORY**: Report Step 1 results to user and announce "Proceeding to Step 2: Risk Analysis"

**Step 2 (MANDATORY): Risk Analysis — Architecture & Task List Verification**
AI must execute:
1. Load `skills/common/risk-discovery-methodology.md`
2. Perform AI self-analysis: simulate each Phase's tasks and identify potential risks
3. Review task lists against To-Be architecture for gaps
4. Present risk findings and recommended checkpoints to user
5. Enter interactive review mode with user (question suggestions provided)
6. Update task lists based on findings
7. **MANDATORY**: Log risk analysis results in `outputs/audit.md`

**⚠️ CRITICAL**: Completing only Step 1 and skipping Step 2 is PROHIBITED. If Step 1 passes, AI must automatically proceed to Step 2 without waiting for user to request it.

### Expected Output
- `outputs/validation/pre-execution-validation-report.md` (covers both Step 1 and Step 2)

### Decision
- **If BLOCKERS**: Stop and report to user
- **If GAPS**: Ask user how to proceed
- **If READY**: Approve to proceed to Stage 6

### Architecture Update Check (MANDATORY)
After validation, check if any findings require changes to the To-Be architecture:
1. Compare validation results against `outputs/architecture/to-be-architecture.md`
2. If changes needed (e.g., new constraints discovered, IaC scope changes):
   - Update `to-be-architecture.md` with changes highlighted
   - Present changes to user for re-approval
   - Do NOT proceed to Stage 6 without re-approval
3. If no changes needed: Confirm architecture remains valid

---

## Stage 6: Task Actions

### When
User provides one of these prompts:
- `prompts/6-task-actions/ecs-path/start-move-to-container-using-ecs-prompt.md`
- `prompts/6-task-actions/start-move-to-container-using-eks-prompt.md`
- `prompts/6-task-actions/start-move-to-managed-database-prompt.md`

### Load These Guides (Safety-Critical)
- `.kiro/guides/stages/execution-stage-guide.md` - Execution rules and patterns
- `.kiro/guides/aws-practices/quality-gates.md` - Quality standards and gates
- `.kiro/guides/common/ai-execution-guidelines.md` - Audit trail logging rules

### During Execution
**Follow these rules**:
1. Execute tasks sequentially (one at a time)
2. **MANDATORY**: Log each task start/completion in `outputs/audit.md` with timestamp
3. Update checkboxes [ ] → [x] after each task
4. Validate after each task (use quality-gates.md)
5. Apply quality gates after each phase
6. Request approval at major phase boundaries

**Quality gates to apply**:
- After containerization: Local test + security scan (MANDATORY)
- After IaC deployment: Dev/staging test (MANDATORY)
- Before production: All tests passed (MANDATORY)

### Error Handling
- Follow error handling from execution-stage-guide.md
- Attempt automatic fix (max 2 attempts)
- Escalate to user if cannot resolve

### Advanced Deployment (Optional)
For EKS cluster setup and ArgoCD GitOps configuration:
- `prompts/6-task-actions/eks-path/define-and-deploy-eks-prompt.md`
- `prompts/6-task-actions/eks-path/setup-argocd-gitops-prompt.md`

---

## Cross-Stage Principles

### Language Detection and Adaptation
**MANDATORY**: Detect the user's language from their first message and apply it throughout the entire workflow.

- **All user-facing output** must be in the detected language:
  - Output documents (analysis, plans, task lists, validation reports)
  - Questions, checkpoint messages, approval requests
  - Error messages and escalation messages
  - Audit log entries in `outputs/audit.md`
- **Section titles and labels** in output templates must be translated to the user's language
- **Internal .kiro documents** (steering, guides, skills) remain in English for global maintainability
- If the user switches language mid-conversation, adapt from that point forward

### Always Apply
Regardless of stage, always:
- Log all actions and decisions in audit.md per `.kiro/guides/common/ai-execution-guidelines.md`
- Reference `.kiro/guides/common/output-templates.md` for consistency

### AI Proactive Behavior (MANDATORY)
AI must act as a proactive guide, not a passive responder:

- **At each stage start**: Proactively present "key items to confirm" before waiting for user input
- **On insufficient input**: Request additional information ("This information is also needed: ...")
- **On potential gaps**: Warn about commonly missed items ("For this workload type, X is usually also considered")
- **On detected patterns**: Suggest relevant follow-up questions based on workload characteristics
- **On missing documents**: Switch to Q&A mode instead of waiting for user to fill templates

### Anti-Skip Rule (MANDATORY)

AI must NOT skip any sub-step within a Stage without explicit user approval.

**Rules**:
1. Each Stage has defined sub-steps (listed in the Stage sections above). ALL sub-steps must be completed in order.
2. If AI determines a sub-step should be skipped (e.g., not applicable), it MUST:
   - Explicitly tell the user which sub-step is being skipped
   - Explain why it believes the sub-step can be skipped
   - Wait for user approval before proceeding
3. Silently skipping a sub-step is PROHIBITED.
4. When a sub-step is completed, AI must announce completion and state the next sub-step.
5. At each sub-step, AI must provide relevant question suggestions to help the user engage effectively.

**Example (correct behavior)**:
```
✅ Step 1 (Automatic Validation) complete. All tools and access permissions verified.

Proceeding to Step 2 (Risk Analysis). Will now verify the To-Be architecture and task lists.

💡 Useful questions to consider at this stage:
- "Will IP-based authentication for external integration systems still work after the transition?"
- "How will in-flight transactions be handled at the cutover point?"
- "Has a cache warm-up strategy been considered?"
```

**Example (prohibited behavior)**:
```
❌ Step 1 complete. Proceeding to Stage 6.
   (Skipped Step 2 without notifying the user)
```

### Configuration Auto-Reference
If `outputs/config.md` exists (created during Stage 1), automatically load and apply saved settings:
- AWS Account ID, Region, Profile → use for all AWS CLI commands
- Git repository URL, branch → use for source code operations
- Do NOT re-ask for settings already saved in config.md

### Quality Standards
- Apply quality gates from `.kiro/guides/aws-practices/quality-gates.md`

### Error Handling
- Detect → Analyze → Fix → Retry → Escalate (if needed)
- Log all errors and resolutions
- Never proceed with unresolved critical errors

### Stage Auto-Progression
AI automatically leads stage-by-stage progression without requiring users to manually load prompt files.

**Session Start — State Detection**:
- Scan `outputs/` directory to determine current progress:
  - `outputs/analysis/as-is-analysis-*.md` exists → Stage 1 complete
  - `outputs/analysis/requirements-analysis.md` exists → Stage 2 complete
  - `outputs/architecture/to-be-architecture.md` exists → Stage 3 complete
  - `outputs/plan/modernization-plan-*.md` exists → Stage 4 complete
  - `outputs/validation/pre-execution-validation-report.md` exists → Stage 5 complete
- **New session (no outputs)**: Guide user through preparation, then start Stage 1
- **Resuming session (outputs exist)**: Report current progress and propose next Stage

**Stage Completion — Auto-Propose Next**:
- When a Stage is complete (output files generated, audit logged), AI reads the next Stage's prompt file from `prompts/` and proposes to proceed
- User confirms with a simple response (e.g., "Please proceed") instead of manually loading prompts
- User can also jump to a specific Stage by directly providing that Stage's prompt

### Context Reset Guidance
When proposing the next Stage, AI should recommend context reset as an **option (not mandatory)**:

- **Explain why**: "If conversation history from previous Stages has accumulated significantly, AI reasoning quality may degrade."
- **State it's optional**: "This is not mandatory, but if you notice response quality declining, we recommend resetting context with the `/clear` command."
- **Explain recovery**: "After resetting, just let me know and I will load the previous Stage's output documents to continue where we left off."
- After `/clear`, AI detects progress from `outputs/` and resumes automatically

### Project Completion & Reuse
When modernization is complete or the user wants to switch to a different project, AI must guide cleanup with explicit user approval.

**Step 1: Confirm completion**
```
All Stage 6 tasks have been completed.
Before cleaning up for reuse, please confirm:
- [ ] All modernization tasks are fully complete
- [ ] Output documents have been saved/exported if needed
- [ ] No further work is needed on this project

Shall I proceed with cleanup? (yes/no)
```

**Step 2: Execute cleanup (ONLY after user approval)**
```bash
/todo delete --all          # Clear TODO list history
rm -rf outputs/             # Remove analysis/plan/validation outputs
rm -rf source-code/         # Remove cloned source code
rm -f plan.md               # Remove working plan file
```

**Step 3: Start fresh**
Exit and restart `kiro-cli chat`, then `/agent swap modernization-agent`

**CRITICAL**: Do NOT execute cleanup without explicit user approval. If the user says "not yet" or "no", do not proceed and continue the current project.

---

## AI Decision Making

### How to Use This Guide
1. **Identify stage**: Read user prompt to determine which stage
2. **Load safety-critical guides**: Load guides listed for that stage
3. **Load skills on demand**: AI automatically discovers relevant skills from `.kiro/skills/`
4. **Execute**: Follow loaded guides to perform work
5. **Validate**: Apply quality gates before proceeding
