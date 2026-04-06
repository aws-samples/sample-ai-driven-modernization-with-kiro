# Healthcare System Modernization Example

This directory contains an example of a healthcare system modernization project.

## Scenario

AWS cloud modernization of a legacy healthcare system

**Current State**:
- Web Portal: 2 EC2 instances (ASP.NET)
- API Server: 2 EC2 instances (.NET Core)
- Database: 1 EC2 instance (SQL Server 2016)
- File Storage: EBS/local storage (medical images)
- Availability: 99%, storage capacity issues and lack of backups

**Target State**:
- AWS cloud migration
- RDS SQL Server (Multi-AZ)
- S3 (medical image storage)
- Full HIPAA compliance
- Availability: 99.9%
- Disaster Recovery (DR) configuration

## Key Features

- **Compliance-Focused**: Full HIPAA compliance
- **Enhanced Security**: Encryption in transit and at rest (KMS)
- **Disaster Recovery**: Cross-Region DR configuration
- **Storage Optimization**: S3 Intelligent-Tiering

## Applicable Workloads

- Healthcare systems
- Picture Archiving and Communication Systems (PACS)
- Electronic Medical Records (EMR/EHR)
- Compliance-critical systems

## Files

- `modernization-requirements.md`: Example requirements specification
