---
name: infrastructure-as-code
description: AWS CDK/Terraform project structure, naming conventions, security checklist (network, security groups, IAM, encryption), high availability, cost optimization. Reference during IaC code development (Stage 3~5).
---
# Infrastructure as Code (IaC) Guide

## IaC Tool Selection

### AWS CDK
**Recommended when**:
- Defining infrastructure with TypeScript/Python
- AWS-centric architecture
- Complex logic required

**Advantages**:
- Leverage programming language features
- Fast support for latest AWS features
- Type safety

### Terraform
**Recommended when**:
- Multi-cloud environments
- Declarative configuration preferred
- Reusing existing modules

**Advantages**:
- Cloud-neutral
- Rich community
- Explicit state management

## Project Structure

### AWS CDK
```
infrastructure/
├── bin/
│   └── app.ts
├── lib/
│   ├── network-stack.ts
│   ├── database-stack.ts
│   ├── container-stack.ts
│   └── pipeline-stack.ts
├── test/
├── cdk.json
└── package.json
```

### Terraform
```
infrastructure/
├── modules/
│   ├── network/
│   ├── database/
│   └── container/
├── environments/
│   ├── dev/
│   └── prod/
├── main.tf
└── variables.tf
```

---

## Stack Separation Guide

### Principle
**Convention first**: If the team already has IaC conventions, follow them. If not, apply the separation guide below to isolate change scope and minimize blast radius.

### Recommended Stack Separation
| Stack | Resources | Change Frequency |
|-------|-----------|-----------------|
| Foundation | VPC, Subnets, NAT Gateway, Security Groups | Rarely |
| Data | RDS/Aurora, ElastiCache, Secrets Manager | Low |
| Compute | ECS/EKS, Task Definitions, Auto Scaling | Medium-High |
| Pipeline | CodePipeline, CodeBuild, ECR | Low |
| CDN | CloudFront, S3 (static assets), WAF | Low |
| Cutover (temporary) | Green ALB, Green Target Groups | Temporary |

### Inter-Stack Communication
- Use CloudFormation Exports/Imports or SSM Parameter Store for loose coupling
- CDK: `stack.exportValue()` / `Fn.importValue()`
- Terraform: `terraform_remote_state` data source or SSM parameters
- Avoid direct cross-stack references that create tight coupling

### Anti-Patterns
- ❌ All resources in a single Stack (any change affects everything)
- ❌ Circular dependencies between Stacks
- ❌ Hardcoded ARNs instead of dynamic references

### Cutover Impact Consideration
When designing Stack separation, consider which Stacks need modification during cutover:
- Ideal: Only Compute + Cutover Stacks change during cutover
- If 3+ Stacks require changes during cutover → ⚠️ review separation strategy

## Naming Conventions

### Resource Naming
```
{project}-{env}-{type}-{purpose}
```

**Examples**:
- `ecommerce-prod-vpc-main`
- `ecommerce-prod-ecs-order`
- `ecommerce-prod-rds-main`

### Required Tags
```typescript
{
  Project: "ecommerce",
  Environment: "prod",
  ManagedBy: "cdk",
  Owner: "devops-team"
}
```

## Environment-Specific Configuration

### Parameter Management
**AWS CDK**:
```typescript
// cdk.json
{
  "context": {
    "dev": {
      "instanceType": "t3.small",
      "minCapacity": 1
    },
    "prod": {
      "instanceType": "t3.large",
      "minCapacity": 2
    }
  }
}
```

### Sensitive Information Management
**Never hardcode secrets — always reference via ARN**

**AWS CDK (Secrets Manager)**:
```typescript
// Reference existing secret
const dbSecret = secretsmanager.Secret.fromSecretNameV2(
  this, 'DBSecret', 'ecommerce/prod/db/credentials'
);

// Inject into ECS Task Definition
container.addSecret('DB_PASSWORD',
  ecs.Secret.fromSecretsManager(dbSecret, 'password')
);
```

**Terraform (Secrets Manager)**:
```hcl
data "aws_secretsmanager_secret" "db" {
  name = "ecommerce/prod/db/credentials"
}

# Reference in ECS task definition
secrets = [{
  name      = "DB_PASSWORD"
  valueFrom = "${data.aws_secretsmanager_secret.db.arn}:password::"
}]
```

**Parameter Store** (for non-sensitive config):
```typescript
const dbHost = ssm.StringParameter.fromStringParameterName(
  this, 'DBHost', '/ecommerce/prod/db/host'
);
```

## Security Checklist

### Network
- [ ] Place applications in private subnets
- [ ] Place databases in DB private subnets
- [ ] Configure NAT Gateway
- [ ] Enable VPC Flow Logs

### Security Groups
- [ ] Least privilege principle
- [ ] No 0.0.0.0/0 (except ALB)
- [ ] Description required

```typescript
const appSG = new ec2.SecurityGroup(this, 'AppSG', {
  vpc,
  description: 'Security group for app containers'
});

appSG.addIngressRule(
  albSG,
  ec2.Port.tcp(8080),
  'Allow from ALB'
);
```

### IAM
- [ ] Least privilege policies
- [ ] Resource-based policies
- [ ] Minimize wildcards (*)

```typescript
taskRole.addToPolicy(new iam.PolicyStatement({
  actions: ['s3:GetObject'],
  resources: [`arn:aws:s3:::${bucketName}/*`]
}));
```

### Encryption
- [ ] Encryption in transit (TLS)
- [ ] Encryption at rest
- [ ] Use KMS keys

```typescript
const database = new rds.DatabaseCluster(this, 'DB', {
  storageEncrypted: true,
  storageEncryptionKey: kmsKey
});
```

## High Availability

### Multi-AZ Deployment
```typescript
const service = new ecs.FargateService(this, 'Service', {
  cluster,
  taskDefinition,
  desiredCount: 2,
  vpcSubnets: {
    subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS
  }
});
```

### Health Check
```typescript
const targetGroup = new elbv2.ApplicationTargetGroup(this, 'TG', {
  vpc,
  port: 8080,
  healthCheck: {
    path: '/health',
    interval: cdk.Duration.seconds(30),
    healthyThresholdCount: 2
  }
});
```

## Cost Optimization

### Fargate Spot
```typescript
const service = new ecs.FargateService(this, 'Service', {
  capacityProviderStrategies: [
    {
      capacityProvider: 'FARGATE_SPOT',
      weight: 2
    },
    {
      capacityProvider: 'FARGATE',
      weight: 1,
      base: 1
    }
  ]
});
```

### Cost Tagging
```typescript
cdk.Tags.of(stack).add('CostCenter', 'engineering');
```

## Pre-Deployment Verification

### CDK
```bash
cdk diff --context environment=prod
```

### Terraform
```bash
terraform plan -var-file=environments/prod/terraform.tfvars
```

### Checklist
- [ ] Security group review
- [ ] IAM permissions minimized
- [ ] Cost impact assessment
- [ ] Backup settings confirmed
- [ ] Tags applied

## Best Practices

### DO
- ✅ Separate environment configurations
- ✅ Use Parameter Store/Secrets Manager
- ✅ Tag all resources
- ✅ Multi-AZ deployment
- ✅ Least privilege principle
- ✅ Enable encryption

### DON'T
- ❌ Hardcoded passwords
- ❌ 0.0.0.0/0 security groups (except ALB)
- ❌ Wildcard IAM permissions
- ❌ Single AZ deployment
- ❌ No encryption
- ❌ Deploy without verification
