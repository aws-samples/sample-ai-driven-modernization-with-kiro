# SaaS Platform Modernization Example

This directory contains an example of a SaaS platform modernization project.

## Scenario

Modernizing Background Workers of a monolithic SaaS application

**Current State**:
- Web Application: 3 EC2 instances (Django)
- Background Workers: 2 EC2 instances (Celery)
- Database: RDS PostgreSQL (single instance)
- Cache: ElastiCache Redis (single node)
- Availability: 98%, Worker overload and DB connection exhaustion issues

**Target State**:
- Worker containerization + Auto Scaling
- Aurora PostgreSQL (Multi-AZ) + RDS Proxy
- Redis cluster mode
- Availability: 99.95%
- Terraform + CI/CD

## Key Features

- **Worker-Focused Modernization**: Optimizing large-scale data processing tasks
- **DB Connection Management**: Resolving connection pool issues with RDS Proxy
- **SQS-Based Auto Scaling**: Dynamic scaling based on queue depth

## Applicable Workloads

- SaaS platforms
- Data processing pipelines
- Asynchronous task processing systems
- Multi-tenant applications

## Files

- `modernization-requirements.md`: Example requirements specification
