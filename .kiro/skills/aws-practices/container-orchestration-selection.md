---
name: container-orchestration-selection
description: Amazon ECS vs EKS selection guide. Decision framework based on team experience, application complexity, operational requirements, and strategic considerations. Reference during Stage 2 requirement analysis.
---
# Container Orchestration Selection Guide

## Purpose
Help decide between Amazon ECS and Amazon EKS for container orchestration.

---

## Decision Framework

### Amazon ECS (Elastic Container Service)

**Best for**:
- ✅ Teams new to containers
- ✅ AWS-native solutions preferred
- ✅ Simpler operational requirements
- ✅ Faster time to production
- ✅ Lower learning curve

**Advantages**:
- Deeply integrated with AWS services
- Simpler to learn and operate
- Lower operational overhead
- Fargate for serverless containers
- Native AWS IAM integration

**Limitations**:
- AWS-specific (not portable)
- Less flexible than Kubernetes
- Smaller ecosystem

**Use when**:
- Team has limited Kubernetes experience
- Need quick deployment
- AWS-only deployment
- Operational simplicity priority

---

### Amazon EKS (Elastic Kubernetes Service)

**Best for**:
- ✅ Teams with Kubernetes expertise
- ✅ Need Kubernetes-specific features
- ✅ Multi-cloud portability required
- ✅ Complex orchestration needs
- ✅ Existing Kubernetes workloads

**Advantages**:
- Industry-standard Kubernetes
- Rich ecosystem and tooling
- Multi-cloud portability
- Advanced features (StatefulSets, DaemonSets, Operators)
- Large community support

**Limitations**:
- Steeper learning curve
- Higher operational complexity
- More configuration required
- Higher costs (control plane + nodes)

**Use when**:
- Team has Kubernetes expertise
- Need advanced orchestration features
- Multi-cloud strategy
- Already using Kubernetes elsewhere

---

## Selection Criteria

### Team Experience
**If team has**:
- No container experience → **ECS** (simpler)
- Docker experience only → **ECS** (easier transition)
- Kubernetes experience → **EKS** (leverage existing skills)
- Both → Consider other factors

### Application Complexity
**If application is**:
- Simple web app → **ECS** (sufficient)
- Microservices (< 10 services) → **ECS** (manageable)
- Complex microservices (> 10) → **EKS** (better orchestration)
- Stateful workloads → **EKS** (StatefulSets)

### Operational Requirements
**If you need**:
- Simple deployment → **ECS**
- Advanced scheduling → **EKS**
- Service mesh → **EKS** (better support)
- Batch jobs → **EKS** (CronJobs)
- DaemonSets → **EKS** (not available in ECS)

### Strategic Considerations
**If you plan**:
- AWS-only → **ECS** (native integration)
- Multi-cloud → **EKS** (portable)
- Rapid scaling → **Both** (both support auto-scaling)
- Cost optimization → **ECS** (Fargate Spot simpler)

---

## Recommendation Process

### Step 1: Assess Team
```markdown
## Question: Kubernetes Experience
Does your team have Kubernetes experience?

A) Yes - Production Kubernetes experience
B) Yes - Some K8s knowledge
C) No - Docker only
D) No - New to containers

[Answer]:
```

### Step 2: Assess Requirements
```markdown
## Question: Orchestration Needs
What orchestration features do you need?

A) Basic - Deploy containers, scale, load balance
B) Moderate - Above + service discovery, secrets
C) Advanced - Above + StatefulSets, DaemonSets, Operators
D) Not sure

[Answer]:
```

### Step 3: Recommend
**Based on answers**:
- A + C or D → **ECS** (team ready, simple needs)
- B + A or B → **EKS** (some experience, moderate needs)
- A + C → **EKS** (advanced features needed)
- C or D + A → **ECS** (no K8s experience, simple needs)

---

## Default Recommendation

**For most modernization projects**: **Amazon ECS**

**Reasons**:
- Simpler to learn and operate
- Faster time to value
- Lower operational overhead
- Sufficient for 80% of use cases
- Easier troubleshooting

**Consider EKS if**:
- Team already knows Kubernetes
- Need specific K8s features
- Multi-cloud strategy
- Complex orchestration requirements

---

## Integration with Workflow

### In Requirement Analysis Stage
Include orchestration selection:
```markdown
## Container Orchestration
- **Selected**: [ECS/EKS]
- **Reason**: [justification based on criteria]
- **Team Experience**: [level]
- **Complexity**: [simple/moderate/complex]
```

### In Planning Stage
Generate appropriate task lists:
- If ECS: Use `prompts/3-make-task-lists/move-to-container-using-ecs-prompt.md`
- If EKS: Use `prompts/3-make-task-lists/move-to-container-using-eks-prompt.md`

### In Execution Stage
Apply appropriate guides:
- ECS: Focus on Task Definitions, Services, Fargate
- EKS: Focus on Deployments, Services, Pods

---

## Migration Path

### Start with ECS, Migrate to EKS Later
**Possible if**:
- Start simple with ECS
- Learn containers in production
- Migrate to EKS when need advanced features
- Use same Docker images

**Migration effort**: Moderate (different orchestration, same containers)
