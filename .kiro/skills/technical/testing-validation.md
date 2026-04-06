---
name: testing-validation
description: Unit/integration/load testing strategies, container image testing, IaC verification, security scanning, pre/post-deployment checklists. Reference during testing and validation tasks (Stage 3~5).
---
# Testing and Validation Guide

## Testing Strategy

### Testing Pyramid
```
        /\
       /E2E\         Few (slow, high cost)
      /------\
     /Integration\   Medium
    /------------\
   /  Unit Tests  \  Many (fast, low cost)
  /----------------\
```

## Unit Tests

### Writing Standards
- Verify independent behavior of each function/method
- Mock external dependencies
- Fast execution speed (milliseconds)
- High coverage target (80%+)

### Example: Backend Unit Test
```
TEST OrderService:
  SETUP:
    mockRepository = Mock(Repository)
    orderService = OrderService(mockRepository)

  TEST "should create order successfully":
    GIVEN orderData = { userId: 1, items: [...] }
    WHEN mockRepository.save returns { id: 1, ...orderData }
    THEN result = orderService.createOrder(orderData)
    ASSERT result.id equals 1
    ASSERT mockRepository.save was called with orderData

  TEST "should throw error when user not found":
    GIVEN mockRepository.findById returns null
    WHEN orderService.createOrder({ userId: 999, items: [] })
    THEN expect error "User not found"
```

### Execution
```bash
# Adjust commands based on test framework
npm test                    # Node.js (Jest/Mocha)
pytest                      # Python (pytest)
go test ./...               # Go
dotnet test                 # .NET
mvn test                    # Java (Maven)
gradle test                 # Java (Gradle)

# Coverage check
npm test -- --coverage      # Node.js
pytest --cov                # Python
go test -cover ./...        # Go
dotnet test /p:CollectCoverage=true  # .NET
```

## Integration Tests

### Writing Standards
- Verify interactions between multiple components
- Use real database (test DB)
- API endpoint testing
- Medium execution speed (seconds)

### Example: API Test
```
TEST Order API:
  SETUP:
    testDb = setupTestDatabase()
    app = createApp(testDb)

  TEARDOWN:
    testDb.close()

  TEST "POST /orders should create order":
    GIVEN request body = { userId: 1, items: [{ productId: 1, quantity: 2 }] }
    WHEN POST to /orders
    THEN response status = 201
    AND response.body has property "id"
    AND response.body.status = "pending"

  TEST "GET /orders/:id should return order":
    GIVEN order exists in testDb
    WHEN GET /orders/{order.id}
    THEN response status = 200
    AND response.body.id = order.id
```

### Database Test
```
TEST Database Operations:
  SETUP:
    connection = createTestConnection()

  CLEANUP after each test:
    TRUNCATE all tables

  TEST "should save and retrieve order":
    GIVEN order = { userId: 1, total: 100 }
    WHEN saved = connection.orders.save(order)
    AND retrieved = connection.orders.findById(saved.id)
    THEN retrieved equals saved
```

## Load Tests

### Purpose
- Identify system performance limits
- Find bottlenecks
- Verify Auto Scaling behavior

### Tool: k6

```
LOAD TEST CONFIGURATION:
  STAGES:
    - Ramp up to 100 users over 2 minutes
    - Stay at 100 users for 5 minutes
    - Ramp up to 200 users over 2 minutes
    - Stay at 200 users for 5 minutes
    - Ramp down to 0 over 2 minutes

  THRESHOLDS:
    - 95% of requests < 500ms
    - Error rate < 1%

  TEST SCENARIO:
    SEND POST request to /orders
    WITH body = { userId: 1, items: [{ productId: 1, quantity: 2 }] }
    CHECK response status = 201
    CHECK response time < 500ms
    WAIT 1 second
    REPEAT
```

**Execution**:
```bash
k6 run load-test.js         # k6
jmeter -n -t test.jmx       # JMeter
locust -f locustfile.py     # Locust (Python)
```

### Performance Criteria
- **Response time**: P95 < 500ms, P99 < 1000ms
- **Throughput**: 100+ requests per second
- **Error rate**: Below 1%
- **CPU utilization**: Below 70% (Auto Scaling trigger)

## Container Tests

### Image Build Test
```bash
# Build image
docker build -t myapp:test .

# Run container
docker run -d -p 8080:8080 --name test-container myapp:test

# Health check
curl http://localhost:8080/health

# Check logs
docker logs test-container

# Cleanup
docker stop test-container
docker rm test-container
```

### Vulnerability Scan
```bash
# Trivy scan
trivy image myapp:test

# Show HIGH severity and above only
trivy image --severity HIGH,CRITICAL myapp:test
```

## Infrastructure Tests

### IaC Verification (CDK)
```bash
# Synth test
cdk synth

# Diff check
cdk diff

# Unit tests
npm test
```

### IaC Verification (Terraform)
```bash
# Format check
terraform fmt -check

# Validation
terraform validate

# Plan check
terraform plan
```

### Security Scan
```bash
# CDK
npm install -g cdk-nag
cdk synth --app "npx ts-node bin/app.ts" --plugin cdk-nag

# Terraform
tfsec .
```

## Pre-Deployment Checklist

### Application
- [ ] Unit tests passed (80%+ coverage)
- [ ] Integration tests passed
- [ ] API endpoints verified
- [ ] Health check endpoint implemented
- [ ] Log output confirmed (stdout/stderr)

### Container
- [ ] Image build successful
- [ ] Vulnerability scan passed (no HIGH+)
- [ ] Container run test passed
- [ ] Health check working
- [ ] Resource limits configured

### Infrastructure
- [ ] IaC syntax validation passed
- [ ] Security scan passed
- [ ] Resource naming conventions followed
- [ ] Tags applied
- [ ] Cost impact reviewed

### Database
- [ ] Backup completed
- [ ] Migration script tested
- [ ] Data validation queries prepared
- [ ] Rollback procedure documented
- [ ] Connection details verified

### Monitoring
- [ ] CloudWatch log groups created
- [ ] Metric collection confirmed
- [ ] Alarms configured
- [ ] Dashboards set up

## Post-Deployment Validation

### Smoke Test
Quick validation of core functionality immediately after deployment:

```bash
#!/bin/bash
# smoke-test.sh

BASE_URL="https://api.example.com"

# Health check
echo "Testing health endpoint..."
curl -f $BASE_URL/health || exit 1

# Create order
echo "Testing create order..."
ORDER_ID=$(curl -X POST $BASE_URL/orders \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"items":[{"productId":1,"quantity":2}]}' \
  | jq -r '.id')

if [ -z "$ORDER_ID" ]; then
  echo "Failed to create order"
  exit 1
fi

# Get order
echo "Testing get order..."
curl -f $BASE_URL/orders/$ORDER_ID || exit 1

echo "Smoke test passed!"
```

### Monitoring Verification
- [ ] Application logs collecting normally
- [ ] CPU/Memory metrics normal
- [ ] Error rate within normal range (< 1%)
- [ ] Response time within normal range (< 500ms)

### Rollback Test
- [ ] Rollback to previous version confirmed possible
- [ ] Rollback time within 5 minutes confirmed

## Best Practices

### DO
- ✅ Automate tests
- ✅ Isolate test environments
- ✅ Fast feedback on failure
- ✅ Run full test suite before deployment
- ✅ Test in production-like environment
- ✅ Define clear performance criteria

### DON'T
- ❌ Skip tests
- ❌ Test directly in production
- ❌ Ignore test failures
- ❌ Skip load testing
- ❌ Deploy without rollback plan
- ❌ Deploy without monitoring
