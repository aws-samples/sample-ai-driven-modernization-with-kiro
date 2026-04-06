# E-Commerce Platform Modernization Example

This directory contains an example of an e-commerce platform modernization project.

## Scenario

Modernizing a legacy 3-tier e-commerce platform to AWS managed services

**Current State**:
- Frontend: 1 EC2 instance (React, Node.js)
- Backend: 1 EC2 instance (Spring Boot, Java 17)
- Database: 1 EC2 instance (MySQL)
- Availability: 95%, incidents during peak traffic

**Target State**:
- Container-based deployment + Auto Scaling
- Managed database (Aurora/RDS)
- Availability: 99.9%
- IaC + CI/CD

## Files

- `modernization-requirements.md`: Example requirements specification
