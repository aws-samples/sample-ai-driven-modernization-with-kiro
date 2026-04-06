# Healthcare System Modernization Requirements Specification

## 1. Current State (As-Is)

### Operating Environment
- **Architecture**: Legacy healthcare system
- **Deployment**: 
  - Web Portal: 2 EC2 instances (ASP.NET, IIS)
  - API Server: 2 EC2 instances (.NET Core)
  - Database: 1 EC2 instance (SQL Server 2016)
  - File Storage: EBS/local storage (medical images)
- **Network**: Single AZ, basic VPC configuration

### Operational Metrics
- **Uptime**: 99% (7 hours downtime per month)
- **Major Incidents**: Storage capacity shortage, lack of backups
- **Deployment**: Quarterly (manual, 4 hours downtime)

---

## 2. Key Pain Points

### P0 (Critical)
1. **Data Security and Compliance**: Insufficient HIPAA compliance, lack of encryption
2. **Storage Capacity Shortage**: Medical image growth reaching storage limits → risk of service disruption

### P1 (Important)
3. **Lack of Backup and Recovery**: Backups exist but no automation or DR site → delayed recovery
4. **Single AZ Deployment**: Service disruption in case of AZ failure

---

## 3. Modernization Goals

### Business Goals
- Improve availability: 99% → 99.9%
- Full HIPAA compliance
- Scalable storage capacity
- Establish disaster recovery (RPO < 1 hour, RTO < 4 hours)

### Technical Goals
1. EC2-based system → AWS managed services
2. SQL Server → Amazon RDS SQL Server (Multi-AZ)
3. Local storage → Amazon S3 (medical images)
4. Enhanced encryption (in transit and at rest)
5. Automated backup and DR configuration

---

## 4. Modernization Scope

### In Scope
- ✅ EC2-based system → AWS managed services
- ✅ SQL Server → RDS SQL Server
- ✅ Local storage → S3 (medical images)
- ✅ Encryption configuration (KMS)
- ✅ Multi-AZ high availability
- ✅ Automated backup and DR
- ✅ VPC and security group configuration

### Out of Scope
- ❌ Application code changes (minimize)
- ❌ Containerization (Phase 2)
- ❌ New feature additions
- ❌ UI/UX improvements

---

## 5. Key Requirements

### 5.1 Database Migration
- SQL Server 2016 → RDS SQL Server 2019
- Multi-AZ configuration
- Automated backup (daily, 35-day retention)
- Encryption (KMS)
- Read Replica (for report generation)

### 5.2 Storage Modernization
- Local storage → S3 (medical images)
- S3 Intelligent-Tiering (cost optimization)
- S3 versioning enabled
- S3 encryption (SSE-KMS)
- CloudFront CDN (image delivery optimization)

### 5.3 Network and Security
- VPC configuration (private subnets)
- Multi-AZ deployment
- Least-privilege security groups
- VPC Flow Logs enabled
- AWS WAF (web application firewall)

### 5.4 Compliance (HIPAA)
- Encryption in transit (TLS 1.2+)
- Encryption at rest (KMS)
- Access logging (CloudTrail, S3 access logs)
- IAM role-based access control
- Regular security audits

### 5.5 Disaster Recovery (DR)
- DR environment in a different region
- RDS automated backup and snapshots
- S3 Cross-Region Replication
- RPO < 1 hour, RTO < 4 hours

### 5.6 Monitoring and Logging
- CloudWatch metrics and alarms
- CloudTrail audit logs
- S3 access logs
- Security event alarms
- Compliance dashboard

---

## 6. Constraints

### Technology Stack
- Compute: Amazon EC2 (Windows Server)
- Database: Amazon RDS SQL Server
- Storage: Amazon S3
- CDN: Amazon CloudFront
- Encryption: AWS KMS
- Network: AWS Direct Connect
- Region: ap-northeast-2 (Seoul), DR: ap-northeast-1 (Tokyo)

### Operational Constraints
- Downtime tolerance: max 4 hours (weekend night)
- Deployment window: weekend night (Saturday 23:00 - Sunday 06:00)
- HIPAA compliance required
- Data backup required (before migration)

### Compliance
- HIPAA (Health Insurance Portability and Accountability Act)
- Personal Information Protection Act
- Medical Service Act

---

## 7. Priority

### P0 (Required)
1. Database migration (RDS SQL Server)
2. Storage migration (S3)
3. Encryption configuration (KMS)

### P1 (Important)
4. Multi-AZ high availability
5. Disaster Recovery (DR) configuration
6. HIPAA compliance

### P2 (Optional)
7. CloudFront CDN
8. Enhanced monitoring

---

## 8. Success Criteria
- [ ] Achieve 99.9% availability
- [ ] Full HIPAA compliance
- [ ] Unlimited storage scalability
- [ ] DR test successful (RPO < 1 hour, RTO < 4 hours)
- [ ] 100% encryption applied

---

## 9. Key Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Data loss | Full backup before migration, use DMS, data validation |
| Compliance violation | Follow HIPAA checklist, leverage AWS Artifact |
| Performance degradation | Load testing, proper instance sizing |
| Budget overrun | Reserved Instances, S3 Intelligent-Tiering |
| Schedule delay | Priority-based execution, weekly checkpoints |
