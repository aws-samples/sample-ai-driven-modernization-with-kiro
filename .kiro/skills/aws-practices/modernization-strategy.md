---
name: modernization-strategy
description: 6R patterns (Rehost/Replatform/Refactor/Repurchase/Retire/Retain), pattern selection criteria, cloud-native target characteristics, iterative approach. Reference during Stage 2~3 strategy planning.
---
# AWS Modernization Strategy

## Six Modernization Patterns

### 1. Rehost (Lift and Shift)
**Description**: Minimal changes, quick migration to cloud
**Timeline**: Weeks to months
**Effort**: Low
**Benefits**: Fast migration, reduced data center costs
**Use when**: Time-sensitive, minimal risk tolerance, quick wins needed

### 2. Replatform (Lift, Tinker, and Shift)
**Description**: Some cloud optimization without major architecture changes
**Example**: MySQL → RDS, add load balancer, implement auto-scaling
**Timeline**: Months
**Effort**: Medium
**Benefits**: Quick cloud benefits with moderate effort
**Use when**: Want cloud benefits without full re-architecture

### 3. Refactor (Re-architect)
**Description**: Cloud-native redesign with significant architecture changes
**Example**: Monolith → Microservices, containers + serverless
**Timeline**: Months to years
**Effort**: High
**Benefits**: Maximum cloud benefits, long-term strategic value
**Use when**: Strategic applications, need full cloud-native capabilities

### 4. Repurchase (Replace)
**Description**: Replace with SaaS solution or commercial product
**Timeline**: Months
**Effort**: Medium
**Benefits**: Reduced maintenance, modern features
**Use when**: Non-differentiating applications, SaaS alternatives available

### 5. Retire
**Description**: Decommission unused or redundant applications
**Timeline**: Weeks
**Effort**: Low
**Benefits**: Cost savings, reduced complexity
**Use when**: Applications with declining business value

### 6. Retain
**Description**: Keep as-is for now, reassess later
**Timeline**: N/A
**Effort**: None
**Benefits**: Defer decision, focus on higher priorities
**Use when**: Applications not ready for modernization

---

## Pattern Selection Criteria

### Business Value Assessment
- Strategic importance: High/Medium/Low
- Revenue impact: Direct/Indirect/None
- Customer impact: Critical/Important/Low
- Competitive advantage: Core/Advantage/Standard/Commodity

### Technical Feasibility
- Cloud readiness: High/Medium/Low
- Technical debt: Low/Moderate/High/Critical
- Complexity: Simple/Moderate/Complex
- Dependencies: Few/Moderate/Many

### Financial Impact
- Current TCO: [amount]
- Migration cost: [estimate]
- Cloud operational cost: [estimate]
- ROI timeframe: [months/years]

### Organizational Readiness
- Team skills: Expert/Intermediate/Beginner
- DevOps maturity: High/Medium/Low
- Change readiness: Ready/Developing/Not ready

---

## Cloud-Native Target Characteristics

### For Refactor/Replatform Patterns
Applications should achieve:
- ✅ **Containerized**: Packaged as lightweight, portable containers
- ✅ **Microservices**: Loosely coupled, independently deployable (if Refactor)
- ✅ **API-Centric**: Well-defined APIs for all interactions
- ✅ **Stateless/Stateful Separation**: Clean separation with externalized state
- ✅ **Infrastructure Independent**: Isolated from server and OS dependencies
- ✅ **Elastic Infrastructure**: Auto-scaling cloud infrastructure
- ✅ **DevOps Integrated**: Agile DevOps processes and automation
- ✅ **Automated Operations**: Built-in automation for deployment, scaling, recovery

---

## Iterative Approach

### Start Small, Learn, Scale
1. **Select 1-2 pilot applications**: Moderate complexity, high business value
2. **Modernize pilots through all phases**: Complete transformation cycle
3. **Learn from experience**: Capture lessons learned
4. **Establish foundation**: Build reusable patterns and expertise
5. **Scale to portfolio**: Apply learnings to remaining applications

### Benefits
- Reduced risk
- Organizational learning
- Proven patterns
- Sustainable transformation

---

## Recommendation Guidelines

### When to recommend Rehost
- Time-sensitive migration
- Low risk tolerance
- Minimal cloud expertise
- Quick data center exit needed

### When to recommend Replatform
- Want cloud benefits quickly
- Moderate effort acceptable
- Some cloud expertise available
- Database or infrastructure optimization primary goal

### When to recommend Refactor
- Strategic application
- Long-term investment
- Need full cloud-native capabilities
- Team has cloud and microservices expertise
- Scalability and agility critical

### When to recommend Repurchase
- Non-differentiating application
- Good SaaS alternatives available
- High maintenance burden
- Limited customization needed

### When to recommend Retire
- Declining business value
- Redundant functionality
- Low usage
- High maintenance cost

### When to recommend Retain
- Not ready for modernization
- Dependencies blocking migration
- Insufficient budget/resources
- Higher priority applications exist
