---
name: container-best-practices
description: Docker multi-stage builds, multi-architecture (buildx), non-root execution, security scanning, Health Check, ECS Task Definition, K8s Deployment and other container best practices. Reference during containerization tasks (Stage 3~5).
---
# Container Best Practices

## Dockerfile Standards

### Multi-Stage Build (MANDATORY)
Separate build dependencies from runtime to minimize image size:

```dockerfile
# Build Stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Runtime Stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 8080
USER node
CMD ["node", "dist/main.js"]
```

---

### Multi-Architecture Build (MANDATORY)

**CRITICAL**: Use docker buildx for multi-architecture support.

**Why required**:
- ✅ Supports both x86_64 (Intel/AMD) and ARM64 (Graviton)
- ✅ AWS Graviton provides 40% better price-performance
- ✅ Single image works on any architecture
- ✅ Future-proof for ARM adoption

**Build command**:
```bash
# Setup buildx (one-time)
docker buildx create --name multiarch --use

# Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:latest \
  --load \
  .
```

**Supported architectures**:
- `linux/amd64`: Intel/AMD 64-bit (standard)
- `linux/arm64`: AWS Graviton, Apple Silicon

**Fallback** (if buildx not available):
```bash
# Standard build (amd64 only)
docker build -t myapp:latest .
echo "⚠️ Built for amd64 only. Install buildx for multi-arch support."
```

---

### Base Image Selection
- Use Alpine Linux (minimal size)
- Use specific version tags (never use latest)
- Prefer official images

```dockerfile
# Good
FROM node:18-alpine

# Bad
FROM node:latest
FROM ubuntu
```

### Layer Caching Optimization
Place infrequently changing commands first:

```dockerfile
# 1. Copy dependency files (low change frequency)
COPY package*.json ./
RUN npm ci

# 2. Copy source code (high change frequency)
COPY . .
RUN npm run build
```

### .dockerignore File
Exclude unnecessary files:

```
node_modules
npm-debug.log
.git
.env
*.md
test/
coverage/
```

## Security Configuration

### Non-root User
Run containers as non-root user:

```dockerfile
# Create and switch user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
USER nodejs

# Or use built-in user
USER node
```

### Vulnerability Scanning
Scan for vulnerabilities after image build:

```bash
# Using Trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:latest

# AWS ECR scan
aws ecr start-image-scan --repository-name myapp --image-id imageTag=latest
```

### Sensitive Information Management
- Inject via environment variables
- Use AWS Secrets Manager/Parameter Store
- Never embed in image

```dockerfile
# Bad
ENV DB_PASSWORD=mysecret

# Good - inject at runtime
# docker run -e DB_PASSWORD=$DB_PASSWORD myapp
```

## Image Optimization

### Size Minimization Techniques
1. Use multi-stage builds
2. Alpine base image
3. Remove unnecessary files
4. Minimize layer count

```dockerfile
# Combine multiple RUN commands into one
RUN apk add --no-cache curl && \
    curl -o /tmp/file.tar.gz https://example.com/file.tar.gz && \
    tar -xzf /tmp/file.tar.gz && \
    rm /tmp/file.tar.gz
```

### Image Size Comparison
```bash
docker images myapp
# REPOSITORY   TAG       SIZE
# myapp        alpine    50MB   ✅
# myapp        ubuntu    200MB  ❌
```

## Health Check Configuration

### Define Health Check in Dockerfile
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

### Application Health Endpoint
```typescript
// Express.js example
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});
```

## Logging Configuration

### Use stdout/stderr
Container logs should output to stdout/stderr:

```typescript
// Good
console.log('Application started');
console.error('Error occurred');

// Bad - writing logs to file
fs.appendFileSync('/var/log/app.log', 'message');
```

### Structured Logging
Output logs in JSON format:

```typescript
console.log(JSON.stringify({
  timestamp: new Date().toISOString(),
  level: 'info',
  message: 'User logged in',
  userId: '12345'
}));
```

## ECS Task Definition

### Resource Limits
Set CPU and memory explicitly:

```json
{
  "family": "myapp",
  "cpu": "512",
  "memory": "1024",
  "containerDefinitions": [{
    "name": "app",
    "image": "myapp:latest",
    "cpu": 512,
    "memory": 1024,
    "essential": true
  }]
}
```

### Environment Variable Management
```json
{
  "environment": [
    {"name": "NODE_ENV", "value": "production"}
  ],
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:region:account:secret:db-password"
    }
  ]
}
```

### Logging Configuration
```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/myapp",
      "awslogs-region": "ap-northeast-2",
      "awslogs-stream-prefix": "ecs"
    }
  }
}
```

## Kubernetes Deployment

### Pod Spec
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: NODE_ENV
          value: production
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
```

## CI/CD Integration

### Image Build and Push
```bash
# Build image
docker build -t myapp:${VERSION} .

# ECR login
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin ${ECR_REGISTRY}

# Tag and push
docker tag myapp:${VERSION} ${ECR_REGISTRY}/myapp:${VERSION}
docker push ${ECR_REGISTRY}/myapp:${VERSION}
```

### Version Management
- Use Git commit SHA
- Semantic versioning
- Never use latest tag

```bash
VERSION=$(git rev-parse --short HEAD)
docker build -t myapp:${VERSION} .
```

## Monitoring

### Metric Collection
- CPU/Memory utilization
- Network I/O
- Container restart count

### CloudWatch Container Insights
```bash
# Enable Container Insights for ECS cluster
aws ecs put-account-setting \
  --name containerInsights \
  --value enabled
```

## Secrets Manager Integration

### ECS Native Integration (Recommended)
Use the `secrets` field in ECS Task Definition to inject secrets at container startup:
```json
"containerDefinitions": [{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:region:account:secret:my-db-secret:password::"
    }
  ]
}]
```
- Secrets are injected as environment variables without embedding in the image
- ECS Task Execution Role needs `secretsmanager:GetSecretValue` permission
- Supports referencing specific JSON keys within a secret

### EKS Integration
- **External Secrets Operator**: Syncs AWS Secrets Manager to Kubernetes Secrets
- **CSI Secrets Store Driver**: Mounts secrets as files in the pod

### Application-Level SDK Access
For dynamic secret retrieval or rotation-aware access:
```
# Application reads secret at runtime via AWS SDK
# Useful when secrets rotate frequently
```

### Secret Rotation
- Enable automatic rotation for DB credentials via Secrets Manager
- Application should handle credential refresh gracefully

---

## Best Practices

### DO
- ✅ Use multi-stage builds
- ✅ Alpine base image
- ✅ Run as non-root user
- ✅ Configure health checks
- ✅ Set resource limits
- ✅ Structured logging
- ✅ Vulnerability scanning
- ✅ Use specific version tags

### DON'T
- ❌ Run as root user
- ❌ Use latest tag
- ❌ Embed sensitive info in image
- ❌ Write logs to filesystem
- ❌ Skip resource limits
- ❌ Include unnecessary packages
- ❌ Skip vulnerability scanning
