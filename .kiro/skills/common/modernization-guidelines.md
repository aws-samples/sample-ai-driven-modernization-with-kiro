---
name: modernization-guidelines
description: Sequential execution, step-by-step validation, production quality principles, security (least privilege, encryption), cost optimization, HA, container/DB/code quality standards. Reference across all Stages.
---
# AI-driven Modernization Guidelines

## Workflow Rules

### 1. Sequential Execution
- ✅ Must complete as-is system analysis before proceeding to design
- ✅ Non-functional requirements must be included in requirement analysis
- ✅ No implementation without risk assessment
- ✅ Validation required after each stage completion

### 2. Planning Before Execution
- Start implementation only after modernization plan and task lists are complete
- Provision infrastructure only after architecture design approval
- Execute database migration only after migration plan is established

### 3. Validation at Every Step
- Validate immediately after each task completion
- Fix issues immediately when discovered
- Confirm current stage completion before proceeding to next

## Infrastructure Modernization

### Quality Standards
- **Production-level quality** required
- Not dev/test level but actual production-ready
- Consider scalability, availability, resilience

### Security Best Practices
- Least Privilege Principle
- Minimal security group openings
- IAM role-based authentication
- Encryption (in-transit and at-rest)
- No hardcoded secrets

### Cost Optimization
- Appropriate resource sizing
- Consider Fargate Spot
- Auto Scaling configuration
- Remove unnecessary resources

### High Availability
- Multi-AZ deployment
- Health Check configuration
- Auto Scaling setup
- Failover planning

## Container Modernization

### Docker Best Practices
- Use multi-stage builds
- Minimal base image (alpine)
- Run as non-root user
- Layer caching optimization
- Create .dockerignore file

### ECS/EKS Deployment
- Clearly define Task Definition / Pod Spec
- Set resource limits (CPU, Memory)
- Configure health check endpoints
- Logging setup (CloudWatch)
- Manage configuration via environment variables

### CI/CD Pipeline
- Source → Build → Deploy basic structure
- Include automated tests
- Pre-deployment approval stage (optional)
- Establish rollback strategy

## Database Modernization

### Data Integrity
- **Data integrity is top priority**
- Validate data before and after migration
- Maintain transaction consistency

### Migration Strategy
- **Full backup before migration is mandatory**
- Validate in test environment first
- Phased transition:
  1. Test environment migration
  2. Staging environment migration
  3. Production environment migration

### Rollback Plan
- Establish rollback procedures in advance
- Retain backup data
- Prepare fast recovery options

### DMS Usage
- Appropriate Replication Instance sizing
- Verify Source/Target Endpoints
- Monitor migration Task
- Execute data validation queries

## Code Quality

### Testing
- Write unit tests
- Perform integration tests
- Load testing (optional)

### Documentation
- Write README files
- Create architecture diagrams
- Document deployment guides
- Write operational runbooks

### Error Handling
- Proper error handling
- Logging and monitoring
- Alarm configuration

## Validation Requirements

### Functional Validation
- Verify feature operation
- Test API endpoints
- Validate data consistency

### Non-Functional Validation
- Performance testing (response time, throughput)
- Security scanning (vulnerability checks)
- Cost verification (confirm expected costs)

### Production Readiness
- Monitoring dashboard configured
- Alarms set up
- Log collection confirmed
- Backup and recovery tested

## AI Tool Usage

### Prompt Best Practices
- Provide clear context
- Specify concrete requirements
- Include constraints
- Define output format

### Validation of AI Output
- Always review AI-generated code
- Directly verify security settings
- Evaluate cost impact
- Sufficient testing before production deployment

### Iterative Improvement
- Improve upon initial results
- Refine requirements through conversation with AI
- Compare multiple options
- Select the best result
