# Audit Rules (MANDATORY)

## Rule
All actions must be logged in `outputs/audit.md`. Create the file if it does not exist.

## When to Log
- Stage/Phase start and completion
- **User prompt input** (original prompt text at stage start and key decision points)
- **User answers to questions** (when user responds to plan.md questions or AI-posed questions)
- File creation or modification
- Command execution (build, deploy, test, etc.)
- Error occurrence and resolution
- Questions asked to user and answers received
- User approval requests and results
- Reason when tests are skipped

## User Input Recording Rule (MANDATORY)
- **Record user input VERBATIM (as-is, unmodified)**. Do not summarize, paraphrase, extract keywords, or transform in any way.
- Use `> ` blockquote format to clearly delineate the original text.
- This applies to: stage-start prompts, answers to questions, approval/rejection responses, direction changes, and any other user input that triggers an action.

## Log Format
```markdown
### [Task Name]
- **Time**: [ISO timestamp]
- **User Prompt** (verbatim):
  > [Paste the user's original input exactly as written, without any modification]
- **Action**: [Action performed]
- **AI Judgment/Suggestion**: [Summary of AI's suggestion or decision]
- **User Answer/Decision** (verbatim):
  > [Paste the user's answer exactly as written, without any modification]
- **Result**: [Success/Failed/Skipped]
- **Files**: [List of created/modified files]
- **Notes**: [Special notes, error details, skip reasons, etc.]
```

## Detailed Guides
- `.kiro/guides/common/ai-execution-guidelines.md` — Execution patterns, checkpoints, error handling
- `.kiro/skills/aws-practices/change-management.md` — Change classification, rollback, risk assessment
