---
name: assessment-framework
description: Five Lenses assessment (strategic/functional/technical/financial/digital readiness), modernization readiness scoring, pattern selection alignment. Reference during Stage 2 requirement analysis.
---
# Assessment Framework

## Five Lenses Assessment

### Purpose
Comprehensive evaluation through five critical dimensions to determine modernization approach.

---

## 1. Strategic/Business Fit

### Assessment Questions
- What is the primary business driver for modernization?
- How critical is this application to business operations?
- Does it provide competitive advantage?
- What is the revenue impact?
- What is the customer impact?

### Evaluation Criteria
- **Business Criticality**: Mission-critical / Important / Supporting / Low impact
- **Competitive Advantage**: Core differentiator / Advantage / Standard / Commodity
- **Revenue Impact**: Direct generation / Enables revenue / Cost center / No impact
- **Customer Impact**: Customer-facing critical / Important / Internal / Back-office

---

## 2. Functional Adequacy

### Assessment Questions
- What are current functional gaps?
- What new capabilities are needed?
- What are integration requirements?
- Are there compliance requirements?

### Evaluation Criteria
- **Current Functionality**: Complete / Adequate / Gaps exist / Major gaps
- **Future Requirements**: Well-defined / Partially defined / Unclear
- **Integration Complexity**: Simple / Moderate / Complex
- **Compliance Needs**: Strict / Moderate / None

---

## 3. Technical Adequacy

### Assessment Questions
- What is the current architecture?
- What is the technical debt level?
- What is the cloud readiness?
- Are there scalability issues?
- Are there security vulnerabilities?

### Evaluation Criteria
- **Architecture**: Modern / Adequate / Legacy / Outdated
- **Technical Debt**: Low / Moderate / High / Critical
- **Cloud Readiness**: High / Medium / Low
- **Scalability**: Scales well / Limited / Cannot scale
- **Security**: Secure / Some issues / Vulnerable

---

## 4. Financial Fit

### Assessment Questions
- What is the current TCO?
- What are estimated migration costs?
- What are projected cloud costs?
- What is the expected ROI?

### Evaluation Criteria
- **Current TCO**: [annual amount]
- **Migration Cost**: [one-time amount]
- **Cloud Operational Cost**: [annual amount]
- **ROI Timeframe**: < 1 year / 1-2 years / 2-3 years / > 3 years
- **Cost Optimization Potential**: High / Medium / Low

---

## 5. Digital Readiness

### Assessment Questions
- What are team cloud skills?
- What is DevOps maturity level?
- What training is needed?
- Is organization ready for change?

### Evaluation Criteria
- **Cloud Skills**: Expert / Intermediate / Beginner / None
- **DevOps Maturity**: High / Medium / Low / None
- **Training Needs**: Minimal / Moderate / Extensive
- **Change Readiness**: Ready / Developing / Not ready

---

## Assessment Scoring

### Overall Readiness Score
Combine all five lenses to determine overall modernization readiness:

**High Readiness** (Proceed with confidence):
- High business value
- Adequate functionality
- Good technical foundation
- Positive financial case
- Team ready

**Medium Readiness** (Proceed with preparation):
- Moderate business value
- Some gaps exist
- Some technical debt
- Acceptable ROI
- Training needed

**Low Readiness** (Defer or prepare):
- Low business value
- Major gaps
- High technical debt
- Poor financial case
- Team not ready

---

## Integration with Current Workflow

### In Requirement Analysis Stage
Include assessment questions:
```markdown
## Strategic Assessment
- Business criticality: [Mission-critical/Important/Supporting/Low]
- Modernization urgency: [High/Medium/Low]
- Expected ROI timeframe: [< 1yr / 1-2yr / 2-3yr / > 3yr]

## Technical Assessment
- Technical debt level: [Low/Moderate/High/Critical]
- Cloud readiness: [High/Medium/Low]
- Scalability: [Good/Limited/Poor]

## Financial Assessment
- Current annual cost: $[amount]
- Estimated migration cost: $[amount]
- Expected cloud cost: $[amount]/year

## Team Readiness
- Cloud skills: [Expert/Intermediate/Beginner/None]
- DevOps maturity: [High/Medium/Low/None]
```

### Use for Pattern Selection
Based on assessment results, recommend appropriate pattern:
- High readiness + High value → Refactor
- Medium readiness + High value → Replatform
- Low readiness + High value → Rehost (then Replatform later)
- Low value → Retire or Retain
