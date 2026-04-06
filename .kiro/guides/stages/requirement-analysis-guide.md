# Requirement Analysis Guide

## Purpose
Guide AI through analyzing modernization requirements based on as-is analysis.

---

## Objectives

### What to Determine
1. Modernization goals (business and technical)
2. Constraints (budget, timeline, downtime)
3. Scope decisions (what to modernize, what to keep)
4. Non-functional requirements (performance, security, availability)
5. Success criteria

---

## Analysis Process

### Step 1: Review As-Is Analysis
- Load all as-is analysis documents
- Understand current state completely
- Identify pain points and limitations

### Step 2: Gather Requirements
Use plan.md to ask:
- Business goals for modernization
- Technical goals (scalability, performance, etc.)
- Budget constraints
- Timeline constraints
- Downtime tolerance
- Compliance requirements
- Team capabilities

### Step 3: Define Scope
- Which components to modernize
- Which components to keep as-is
- Integration points to maintain
- New capabilities to add

### Step 4: Define NFRs
- Performance targets (response time, throughput)
- Availability targets (uptime %)
- Scalability requirements (users, data volume)
- Security requirements (encryption, compliance)
- Disaster recovery (RTO, RPO)

---

## Output Format

**File**: `outputs/analysis/requirements-analysis.md`

```markdown
# Requirements Analysis

## Business Goals
- [Goal 1]: [description and success metric]
- [Goal 2]: [description and success metric]

## Technical Goals
- [Goal 1]: [description and target]
- [Goal 2]: [description and target]

## Constraints
- **Budget**: [amount or range]
- **Timeline**: [duration]
- **Downtime**: [acceptable hours]
- **Team**: [size and skills]

## Scope Definition

### In Scope
- [Component 1]: [what will be modernized]
- [Component 2]: [what will be modernized]

### Out of Scope
- [Component]: [reason for exclusion]

### Integration Points
- [Integration]: [how it will be handled]

## Non-Functional Requirements

### Performance
- Response time: [target]
- Throughput: [requests/second]
- Concurrent users: [number]

### Availability
- Uptime target: [%]
- Multi-AZ: [Yes/No]
- Disaster recovery: RTO [hours], RPO [hours]

### Scalability
- Auto-scaling: [Yes/No]
- Min/Max instances: [numbers]
- Growth projection: [%/year]

### Security
- Encryption: [at-rest/in-transit]
- Compliance: [standards]
- Authentication: [method]
- Network isolation: [requirements]

## Success Criteria
- [ ] [Measurable criterion 1]
- [ ] [Measurable criterion 2]
- [ ] [Measurable criterion 3]

## Risks
- **Risk**: [description] - **Mitigation**: [plan]
```

---

## Validation

### Requirements Validation
- [ ] Business goals clearly defined
- [ ] Technical goals measurable
- [ ] Constraints realistic
- [ ] Scope clearly bounded
- [ ] NFRs specific and testable
- [ ] Success criteria measurable
- [ ] Risks identified with mitigations

### Completeness Check
- [ ] All questions answered
- [ ] No assumptions without confirmation
- [ ] Specific values (not "TBD")
- [ ] Ready for planning stage
