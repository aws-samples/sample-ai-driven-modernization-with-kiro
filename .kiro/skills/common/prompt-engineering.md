---
name: prompt-engineering
description: Question/Answer pattern, plan.md process, role definition, scope limitation, output format specification, task list format. Reference for user interaction and planning across all Stages.
---
# AI Prompt Engineering Guide

## Role Definition

### Clear Role Setting
Define the AI agent's role clearly at the start of each prompt.

```markdown
Your role: You are a [role name] for the workload modernization project, responsible for [task description].
```

**Examples**:
- Application Development Engineer
- DBA (Database Administrator)
- Infrastructure Engineer
- Project Manager
- Full-stack Engineer

## Information Gathering Patterns

### Question/Answer Pattern
Use a structure where AI asks questions and users fill in answers.

```markdown
[Question]
Write your question here.

[Answer]
(Space for user to write answer)
```

**When to use**:
- When code repository URL is needed
- When DB connection info is needed
- When AWS account info is needed
- When technology stack selection is needed

## Planning Process

### 1. Create Plan Document
Instruct AI to write a step-by-step plan in `plan.md`.

```markdown
Write the required information and step-by-step plan in plan.md.
```

### 2. No Self-Decisions
Explicitly state that AI should not make assumptions or decisions on its own.

```markdown
Do not make assumptions or decisions on your own.
After writing the plan, request my review and approval.
```

### 3. Step-by-Step Execution
Instruct to execute one step at a time after approval.

```markdown
After receiving my approval, you may execute the plan one step at a time.
Mark the checkbox as complete after finishing each step.
```

## Scope Limitation

### Specify Focus Area
Limit AI to focus on specific areas only.

```markdown
Focus only on [specific area] analysis and do not work on anything else.
Detailed analysis of [other areas] will be done in separate stages.
```

**Examples**:
- "Focus only on application analysis. Infrastructure and database will be analyzed in separate stages."
- "Focus only on database analysis. Application and infrastructure will be analyzed in separate stages."

## Output Format Specification

### Specify Output Directory and Filename
Clearly specify where to save results and the filename.

```markdown
When work is complete, create [filename].md in the './outputs/[directory]/' directory.
```

**Examples**:
- `./outputs/analysis/as-is-analysis-application.md`
- `./outputs/plan/modernization-plan-ecs.md`
- `./outputs/tasks/ecs/1. Foundation Infrastructure Tasks.md`

## Task List Format

### Use Todo-list Format
Instruct to write task lists with checkbox format.

```markdown
Write each task in Todo-list format using `[ ]` markers.
Completed tasks will be marked with `[x]`.
```

### Consider Task Ordering
Instruct to arrange tasks considering dependencies.

```markdown
Arrange each stage in the order: `task → test and verify → proceed to next stage`.
When writing task lists, arrange task order considering dependencies between tasks.
```

## Reference Document Specification

### Provide Clear Document Paths
Clearly specify documents that AI should reference.

```markdown
Perform [task name] referencing the following documents:
- /outputs/analysis/as-is-analysis-infrastructure.md
- /outputs/analysis/as-is-analysis-application.md
- /outputs/analysis/requirements-analysis.md
```

## Completion Confirmation Message

### Guide User Actions
Instruct AI to guide users on next actions after work completion.

```markdown
After all answers are written, tell me to type: 'I have answered all questions for [task name] in plan.md.
Please review the plan and add Questions to plan.md if you have additional questions.'
```

## Prompt Structure Template

```markdown
Your role: [Role definition]

Perform [task name] referencing the following documents:
- [Document path 1]
- [Document path 2]

* * *

Write the required information and step-by-step plan in plan.md.
For parts that need clarification, ask with [Question] and [Answer] tags.

Do not make assumptions or decisions on your own.
After writing the plan, request my review and approval.

After receiving my approval, you may execute the plan one step at a time.
Mark the checkbox as complete after finishing each step.

After all answers are written, tell me to type: 'I have answered all questions for [task name] in plan.md.
Please review the plan and add Questions to plan.md if you have additional questions.'

[Scope limitation description]

When work is complete, create [filename].md in the './outputs/[directory]/' directory.
```

## Best Practices

### DO
- ✅ Define roles clearly
- ✅ Request information in structured format
- ✅ Include step-by-step approval process
- ✅ Specify output format and location
- ✅ Clearly limit scope
- ✅ Provide reference document paths

### DON'T
- ❌ Vague instructions ("do it well", "handle appropriately")
- ❌ Request multiple tasks at once
- ❌ Unspecified output format
- ❌ Allow unrestricted work without scope limits
- ❌ Skip validation steps

---

## Suggested Questions by Stage

When users are unsure what to ask, AI should suggest effective questions. This section serves as a reference for both AI and users.

### Stage 1: As-Is Analysis
- "What happens when the application receives SIGTERM? Does it drain connections?"
- "Are there any long-running background tasks (workers, schedulers, batch jobs)?"
- "What security policies or network restrictions exist in the current environment?"
- "Are there existing AWS resources (VPC, ALB, ECS cluster) that should be reused?"

### Stage 2: Requirement Analysis
- "What is the maximum acceptable downtime during migration?"
- "Are there compliance requirements (HIPAA, PCI-DSS, etc.) that affect architecture choices?"
- "What is the expected traffic growth over the next 12 months?"

### Stage 3: To-Be Architecture Design
- "Why was this specific architecture pattern chosen over alternatives?"
- "What happens if this component fails? Is there a fallback?"
- "How does this design handle the [specific constraint] we identified?"

### Stage 4: Task List Generation
- "Are there tasks that can run in parallel to save time?"
- "What is the rollback plan if this task fails?"
- "Is there a dependency I'm missing between these tasks?"

### Stage 5: Pre-Execution Validation
- "What permissions are needed for each tool?"
- "Are there any environment-specific blockers we haven't checked?"

### Stage 6: Task Execution
- "Can you explain what this command does before running it?"
- "What should I verify after this task completes?"
- "What are the signs that something went wrong?"
