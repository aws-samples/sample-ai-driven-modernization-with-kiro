# Example Projects

This directory contains examples of various workload modernization scenarios.

## Included Examples

### 1. E-Commerce Platform (`ecommerce/`)
Modernizing a legacy 3-tier web application to container-based + managed database

**Key Features**:
- Monolithic → Microservice separation (Order Service)
- EC2 → Container orchestration (ECS/EKS)
- EC2 MySQL → Aurora/RDS
- Manual deployment → CI/CD pipeline

**Applicable Workloads**:
- E-commerce platforms
- Reservation systems
- Online marketplaces

### 2. SaaS Platform (`saas-platform/`)
Modernizing Background Workers of a monolithic SaaS application

**Key Features**:
- Worker containerization + Auto Scaling
- RDS PostgreSQL → Aurora PostgreSQL + RDS Proxy
- Redis cluster mode migration
- SQS-based Auto Scaling

**Applicable Workloads**:
- SaaS platforms
- Data processing pipelines
- Asynchronous task processing systems
- Multi-tenant applications

### 3. Healthcare System (`healthcare-system/`)
AWS cloud modernization of a legacy healthcare system

**Key Features**:
- Database modernization (SQL Server → RDS)
- Storage modernization (local storage → S3)
- HIPAA compliance
- Disaster Recovery (DR) configuration

**Applicable Workloads**:
- Healthcare systems
- Picture Archiving and Communication Systems (PACS)
- Electronic Medical Records (EMR/EHR)
- Compliance-critical systems

---

## How to Use Examples

1. Select a similar scenario
2. Refer to `modernization-requirements.md` to write your own requirements
3. Execute the prompt workflow

## Adding New Examples

To add examples for other scenarios:
1. Create a new directory (e.g., `fintech-platform/`)
2. Write `modernization-requirements.md`
3. Add scenario description to `README.md`
4. Add the example to this file
