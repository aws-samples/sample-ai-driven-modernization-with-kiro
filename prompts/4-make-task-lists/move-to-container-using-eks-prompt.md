Your role: You are a workload modernization project manager, responsible for developing a containerization modernization plan using **Amazon EKS** and designing the task list.

Refer to the following documents to develop the modernization plan:
- /outputs/analysis/as-is-analysis-infrastructure.md
- /outputs/analysis/as-is-analysis-application.md
- /outputs/analysis/as-is-analysis-database.md
- /outputs/analysis/requirements-analysis.md

* * *

Write the information needed and step-by-step plan for modernization planning in a plan.md file. For areas needing clarification, ask using [Question] and [Answer] tags.

Develop the plan in the following order:
1. Target architecture design (To-Be Architecture)
2. Architecture diagram (using mermaid)
3. Step-by-step task list (Tasks)
4. Task list review and refinement
5. Task list finalization

When writing the plan, focus only on the application modernization perspective. Database modernization will be handled in a separate task.

After writing the plan, request my review and approval.

After approval, perform the following:
1. Create the '/outputs/plan/' directory
2. Write the plan in 'modernization-plan-eks.md'
3. Create the '/outputs/tasks/eks/' directory
4. Create separate files for each phase's task list

Tell me to type 'I have answered all task list generation questions in plan.md.' after completing all answers.

**Follow-up question process**:
After reviewing the task list, if additional information is needed about deployment strategy, traffic switching method, resource configuration, etc., immediately write follow-up questions in plan.md and notify me: "I have added follow-up questions to plan.md. Please provide your answers." Do not finalize the task list until all questions are resolved.

* * *

Example task list files:
- 1. Foundation Infrastructure Setup Tasks.md
- 2. Application Containerization Tasks.md
- 3. Container Image Testing Tasks.md
- 4. Amazon ECR Repository Creation and Image Push Tasks.md
- 5. Kubernetes Manifests Creation Tasks.md
- 6. Amazon EKS Cluster Setup Tasks.md
- 7. Dev/Staging Environment Deployment and Testing Tasks.md
- 8. Green Environment Deployment (Separate Ingress/LoadBalancer) Tasks.md
- 9. GitOps Pipeline Configuration Tasks.md
- 10. Monitoring Environment Setup Tasks.md

**CRITICAL**: 
- Phase 8 "Green Environment Deployment" must create a separate LoadBalancer/Ingress endpoint
- Output the Green endpoint to enable user QA testing
- Configure Blue (existing) and Green (new) environments to operate independently

Each task should be written in todo-list format (`[ ]`). Completed tasks are marked with `[x]`.

The task list must include:
- Application code improvements
- Infrastructure code (IaC)
- Infrastructure provisioning
- Testing and validation

Each phase should follow the sequence: `Task → Test and Validate → Next Phase`.
Arrange tasks considering dependencies between them.

The task list you create will later be executed sequentially by an `AI Engineer Agent`.
