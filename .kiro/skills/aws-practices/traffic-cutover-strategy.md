---
name: traffic-cutover-strategy
description: Canary/Blue-Green/Rolling deployment strategies, ALB Weighted Target Groups, monitoring metrics, rollback triggers, cutover script templates. Reference during Stage 5 traffic cutover policy planning.
---
# Traffic Cutover Strategy

## Purpose
Safe, gradual traffic transition from old (Blue) to new (Green) environment.

---

## Cutover Strategies

### Strategy 1: Canary Deployment (Recommended)
**Gradual traffic shift with validation at each step**

```
Phase 1: 5% → Green   (30 min monitoring)
Phase 2: 20% → Green  (1 hour monitoring)
Phase 3: 50% → Green  (2 hour monitoring)
Phase 4: 100% → Green (24 hour monitoring)
```

**Advantages**:
- ✅ Minimal risk (small traffic first)
- ✅ Easy rollback at any phase
- ✅ Real production validation
- ✅ Issues affect small percentage

**Use when**: Default for most cases

---

### Strategy 2: Blue-Green (All at once)
**Instant switch with immediate rollback capability**

```
Before: Blue 100% → Green 0%
After:  Blue 0%   → Green 100%
```

**Advantages**:
- ✅ Fast cutover
- ✅ Instant rollback
- ✅ Simple process

**Use when**: 
- High confidence in Green
- Extensive QA completed
- Need fast cutover

---

### Strategy 3: Rolling (No extra resources)
**Replace instances one by one**

```
Instance 1: Blue → Green
Instance 2: Blue → Green
Instance 3: Blue → Green
```

**Advantages**:
- ✅ No extra resources needed
- ✅ Gradual transition

**Disadvantages**:
- ⚠️ Slower rollback
- ⚠️ Mixed versions during rollout

---

## Implementation Methods

### Method 1: ALB Weighted Target Groups (Recommended)
```typescript
// Canary: 5% to Green
listener.addAction('weighted', {
  action: elbv2.ListenerAction.weightedForward([
    { targetGroup: blueTargetGroup, weight: 95 },
    { targetGroup: greenTargetGroup, weight: 5 }
  ])
});
```

**Advantages**:
- Easy to adjust weights
- Instant changes
- Built-in health checks

---

### Method 2: Route53 Weighted Routing
```
Blue ALB: Weight 95
Green ALB: Weight 5
```

**Advantages**:
- DNS-level control
- Works across regions
- Independent ALBs

**Disadvantages**:
- DNS caching delays
- Slower changes

---

### Method 3: Target Group Replacement on Existing ALB (Recommended for Green ALB → Production ALB migration)

When a Green ALB was used for testing and the workload needs to move to the existing production ALB:

**Problem**: Moving a Target Group from Green ALB to existing ALB requires detaching the listener from Green ALB first, which creates a gap.

**Recommended approach**: Create a new Target Group on the existing ALB instead of moving the Green ALB's TG.

```
Step 1: Create new TG on existing ALB (container workload targets)
Step 2: Register container tasks/instances to new TG
Step 3: Verify health checks pass on new TG
Step 4: Update existing ALB listener to forward to new TG
Step 5: Remove old TG (EC2-based) from listener
Step 6: Decommission Green ALB (after rollback period)
```

**Advantages**:
- No downtime during transition
- Green ALB remains intact as rollback target during transition
- Clean separation between test (Green ALB) and production (existing ALB)

---

## Monitoring During Cutover

### Critical Metrics
- **Error Rate**: Must stay < 1%
- **Response Time**: P95 < 500ms, P99 < 1000ms
- **Availability**: > 99.9%
- **CPU/Memory**: < 70% utilization

### Rollback Triggers
**Automatic rollback if**:
- Error rate > 5%
- Response time P95 > 2000ms
- Health check failures > 10%
- Critical errors in logs

**Manual rollback if**:
- Business impact detected
- Data inconsistencies
- User complaints spike
- Any critical issue

---

## Cutover Scripts

### Script 1: cutover-phase1-canary-5.sh
```bash
#!/bin/bash
# Shift 5% traffic to Green

echo "Current state:"
# Show current traffic distribution

echo "Will change to: Blue 95%, Green 5%"
read -p "Proceed? (yes/no): " confirm

if [ "$confirm" = "yes" ]; then
  # Update ALB listener
  aws elbv2 modify-listener ...
  
  echo "✅ Phase 1 complete"
  echo "Monitor for 30 minutes"
  echo "Next: ./cutover-phase2-progressive-20.sh"
fi
```

### Script 2: rollback-to-blue.sh
```bash
#!/bin/bash
# Emergency rollback to Blue

echo "⚠️ ROLLBACK: Shifting 100% traffic to Blue"
read -p "Confirm rollback? (yes/no): " confirm

if [ "$confirm" = "yes" ]; then
  # Immediate switch to Blue
  aws elbv2 modify-listener ...
  
  echo "✅ Rollback complete"
  echo "All traffic back to Blue environment"
fi
```

### Script 3: verify-traffic.sh
```bash
#!/bin/bash
# Verify current traffic distribution

echo "=== Traffic Distribution ==="
# Query ALB listener configuration

echo "=== Health Status ==="
# Check target group health

echo "=== Recent Metrics ==="
# Show error rate, response time
```

---

## Cutover Checklist

### Before Starting Cutover
- [ ] Green environment fully deployed
- [ ] All tests passed
- [ ] Customer QA approved
- [ ] Monitoring configured
- [ ] Rollback scripts tested
- [ ] Team ready to monitor

### During Each Phase
- [ ] Execute cutover script
- [ ] Wait monitoring period
- [ ] Check all metrics
- [ ] Verify no errors
- [ ] Confirm ready for next phase

### After Full Cutover
- [ ] 100% traffic on Green
- [ ] Blue environment on standby
- [ ] Monitor for 24-48 hours
- [ ] Verify stability
- [ ] Plan Blue environment cleanup

---

## Rollback Procedures

### When to Rollback
- Error rate exceeds threshold
- Performance degradation
- Data issues detected
- Business impact
- Any critical problem

### How to Rollback
1. Execute: `./rollback-to-blue.sh`
2. Verify: `./verify-traffic.sh`
3. Confirm: All traffic back to Blue
4. Investigate: Analyze Green environment issues
5. Fix and redeploy Green
6. Retry cutover when ready

---

## Post-Cutover Rollback Period (MANDATORY for Production)

After full cutover, maintain rollback capability for a defined period (recommended: 1 week).

### Resources to Retain During Rollback Period
| Resource Type | Action | Purpose |
|--------------|--------|---------|
| EC2 instances | **Stop** (do not terminate) | Quick restore by starting instances |
| AMI/Snapshot | Create via AWS Backup or manual AMI | Restore point if instances are needed |
| Source DB | Keep read-only or standby | Data fallback if issues discovered |
| Green ALB | Keep active (no traffic) | Rollback target for traffic switch |
| Old Target Groups | Keep registered | Instant rollback by switching listener |

### Rollback Period Checklist
- [ ] Rollback period duration agreed with user (default: 1 week)
- [ ] EC2 instances stopped (not terminated)
- [ ] AMI/snapshots created for critical instances
- [ ] Source DB maintained in read-only or standby mode
- [ ] Monitoring active on new environment for the full rollback period
- [ ] Rollback procedure documented and tested

### Post-Rollback Period Cleanup
After rollback period expires without issues:
- [ ] Terminate stopped EC2 instances
- [ ] Delete old AMIs/snapshots (retain per backup policy)
- [ ] Decommission Green ALB and old Target Groups
- [ ] Remove old DB replicas/standby
- [ ] Update documentation to reflect final state

---

## Best Practices

### DO
- ✅ Start with small percentage (5%)
- ✅ Monitor closely at each phase
- ✅ Wait sufficient time between phases
- ✅ Have rollback script ready
- ✅ Keep Blue environment running
- ✅ Document all changes

### DON'T
- ❌ Rush through phases
- ❌ Skip monitoring periods
- ❌ Ignore warning signs
- ❌ Delete Blue immediately
- ❌ Cutover during peak hours
- ❌ Proceed with unresolved issues
